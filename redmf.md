using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Runtime.InteropServices;
using System.Text;
using System.Threading.Tasks;

public sealed class HighEndMemory : IDisposable
{
    #region Win32 API
    [DllImport("kernel32.dll", SetLastError = true)]
    static extern IntPtr OpenProcess(uint dwDesiredAccess, bool bInheritHandle, int dwProcessId);

    [DllImport("kernel32.dll", SetLastError = true)]
    static extern bool ReadProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, [Out] byte[] lpBuffer, int dwSize, out int lpNumberOfBytesRead);

    [DllImport("kernel32.dll", SetLastError = true)]
    static extern bool WriteProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, byte[] lpBuffer, int dwSize, out int lpNumberOfBytesWritten);

    [DllImport("kernel32.dll", SetLastError = true)]
    static extern bool VirtualQueryEx(IntPtr hProcess, IntPtr lpAddress, out MEMORY_BASIC_INFORMATION lpBuffer, uint dwLength);

    [DllImport("kernel32.dll", SetLastError = true)]
    static extern bool CloseHandle(IntPtr hObject);

    [DllImport("psapi.dll", SetLastError = true)]
    static extern bool EnumProcessModules(IntPtr hProcess, [Out] IntPtr[] lphModule, uint cb, out uint lpcbNeeded);

    [DllImport("psapi.dll", SetLastError = true)]
    static extern bool GetModuleBaseName(IntPtr hProcess, IntPtr hModule, [Out] StringBuilder lpBaseName, uint nSize);

    [DllImport("psapi.dll", SetLastError = true)]
    static extern bool GetModuleInformation(IntPtr hProcess, IntPtr hModule, out MODULEINFO lpmodinfo, uint cb);

    const uint PROCESS_ALL_ACCESS = 0x1F0FFF;
    const uint MEM_COMMIT = 0x1000;

    [StructLayout(LayoutKind.Sequential)]
    struct MEMORY_BASIC_INFORMATION
    {
        public IntPtr BaseAddress;
        public IntPtr AllocationBase;
        public uint AllocationProtect;
        public IntPtr RegionSize;
        public uint State;
        public uint Protect;
        public uint Type;
    }

    [StructLayout(LayoutKind.Sequential)]
    struct MODULEINFO
    {
        public IntPtr lpBaseOfDll;
        public uint SizeOfImage;
        public IntPtr EntryPoint;
    }
    #endregion

    IntPtr _handle;
    readonly object _regionLock = new object();
    List<MemRegion> _cachedRegions;
    DateTime _regionCacheTime = DateTime.MinValue;
    readonly TimeSpan _regionTtl = TimeSpan.FromSeconds(5);

    public bool OpenProcess(string name)
    {
        var p = Process.GetProcessesByName(name);
        return p.Length != 0 && OpenProcess(p[0].Id);
    }

    public bool OpenProcess(int pid)
    {
        _handle = OpenProcess(PROCESS_ALL_ACCESS, false, pid);
        _cachedRegions = null;
        return _handle != IntPtr.Zero;
    }

    #region Module Base
    public long GetModuleBase(string moduleName)
    {
        IntPtr[] mods = new IntPtr[1024];
        if (!EnumProcessModules(_handle, mods, (uint)(mods.Length * IntPtr.Size), out uint cbNeeded))
            return 0;

        int count = (int)(cbNeeded / IntPtr.Size);
        StringBuilder name = new StringBuilder(1024);
        for (int i = 0; i < count; i++)
        {
            if (GetModuleBaseName(_handle, mods[i], name, 1024) > 0)
                if (name.ToString().Equals(moduleName, StringComparison.OrdinalIgnoreCase))
                    return (long)mods[i];
        }
        return 0;
    }

    public uint GetModuleSize(string moduleName)
    {
        IntPtr[] mods = new IntPtr[1024];
        if (!EnumProcessModules(_handle, mods, (uint)(mods.Length * IntPtr.Size), out uint cbNeeded))
            return 0;

        int count = (int)(cbNeeded / IntPtr.Size);
        StringBuilder name = new StringBuilder(1024);
        for (int i = 0; i < count; i++)
        {
            if (GetModuleBaseName(_handle, mods[i], name, 1024) > 0)
                if (name.ToString().Equals(moduleName, StringComparison.OrdinalIgnoreCase))
                {
                    if (GetModuleInformation(_handle, mods[i], out MODULEINFO info, (uint)Marshal.SizeOf(typeof(MODULEINFO))))
                        return info.SizeOfImage;
                    return 0;
                }
        }
        return 0;
    }
    #endregion

    #region Pointer Chain
    public long ReadPointerChain(long baseAddress, int[] offsets)
    {
        long addr = baseAddress;
        for (int i = 0; i < offsets.Length - 1; i++)
        {
            addr = ReadMemory<long>(addr + offsets[i]);
            if (addr == 0) throw new Exception("Null pointer in chain");
        }
        return addr + offsets[offsets.Length - 1];
    }

    public bool TryReadPointerChain(long baseAddress, int[] offsets, out long result)
    {
        result = 0;
        try
        {
            result = ReadPointerChain(baseAddress, offsets);
            return true;
        }
        catch { return false; }
    }
    #endregion

    #region Memory Regions
    List<MemRegion> GetRegions(bool writable, bool executable)
    {
        lock (_regionLock)
        {
            if (_cachedRegions != null && DateTime.Now - _regionCacheTime < _regionTtl)
                return _cachedRegions.FindAll(r => r.Size > 0);

            var list = new List<MemRegion>();
            IntPtr addr = IntPtr.Zero;
            uint sz = (uint)Marshal.SizeOf(typeof(MEMORY_BASIC_INFORMATION));

            while (VirtualQueryEx(_handle, addr, out MEMORY_BASIC_INFORMATION mbi, sz))
            {
                if (mbi.State == MEM_COMMIT)
                {
                    bool isW = (mbi.Protect & 0x04) != 0 || (mbi.Protect & 0x40) != 0;
                    bool isX = (mbi.Protect & 0x20) != 0 || (mbi.Protect & 0x40) != 0 || (mbi.Protect & 0x10) != 0;

                    if ((!writable || isW) && (!executable || isX))
                    {
                        long rs = (long)mbi.RegionSize;
                        if (rs > int.MaxValue) rs = int.MaxValue;
                        list.Add(new MemRegion { Base = mbi.BaseAddress, Size = (int)rs });
                    }
                }
                long nxt = (long)mbi.BaseAddress + (long)mbi.RegionSize;
                if (nxt <= (long)addr) break;
                addr = (IntPtr)nxt;
            }

            _cachedRegions = list;
            _regionCacheTime = DateTime.Now;
            return list.FindAll(r => r.Size > 0);
        }
    }

    struct MemRegion { public IntPtr Base; public int Size; }
    #endregion

    #region AoB Scan
    public Task<List<long>> AoBScan(string signature, bool writable, bool executable)
    {
        return Task.Run(() => AoBScanSync(signature, writable, executable));
    }

    List<long> AoBScanSync(string signature, bool writable, bool executable)
    {
        var pat = ParseAoB(signature);
        if (pat.Length == 0) return new List<long>();

        var regions = GetRegions(writable, executable)
                      .FindAll(r => r.Size >= pat.Length);

        var hits = new ConcurrentBag<long>();

        Parallel.ForEach(regions, new ParallelOptions { MaxDegreeOfParallelism = Environment.ProcessorCount }, reg =>
        {
            byte[] localBuf = new byte[4 * 1024 * 1024];
            ScanRegion(reg.Base, reg.Size, pat, hits, localBuf);
        });

        return hits.OrderBy(x => x).ToList();
    }

    void ScanRegion(IntPtr baseAddr, int size, AoBPattern p, ConcurrentBag<long> hits, byte[] buffer)
    {
        int patLen = p.Length;
        int anchorOff = p.AnchorOffset;
        byte[] anchor = p.AnchorSequence;
        int anchorLen = anchor.Length;
        bool bruteForce = anchorLen == 0;
        int pos = 0;

        while (pos < size)
        {
            int toRead = Math.Min(buffer.Length, size - pos);
            if (toRead < patLen) break;

            IntPtr readAddr = IntPtr.Add(baseAddr, pos);
            if (!ReadProcessMemory(_handle, readAddr, buffer, toRead, out int bytesRead) || bytesRead < patLen)
            {
                pos += toRead;
                continue;
            }

            if (bruteForce)
            {
                int limit = bytesRead - patLen + 1;
                for (int i = 0; i < limit; i++)
                {
                    bool ok = true;
                    for (int j = 0; j < patLen; j++)
                        if (p.Mask[j] && buffer[i + j] != p.Bytes[j]) { ok = false; break; }
                    if (ok) hits.Add((long)readAddr + i);
                }
            }
            else
            {
                int search = 0;
                int limit = bytesRead - patLen + 1;

                while (search < limit)
                {
                    int idx = IndexOfBytes(buffer, search, limit - search, anchor);
                    if (idx < 0) break;
                    int abs = search + idx;
                    int start = abs - anchorOff;

                    if (start >= 0 && start + patLen <= bytesRead)
                    {
                        bool ok = true;
                        for (int j = 0; j < patLen; j++)
                            if (p.Mask[j] && buffer[start + j] != p.Bytes[j]) { ok = false; break; }
                        if (ok) hits.Add((long)readAddr + start);
                    }
                    search = abs + 1;
                }
            }

            pos += bytesRead - patLen + 1;
        }
    }

    static int IndexOfBytes(byte[] data, int start, int maxLen, byte[] seq)
    {
        int seqLen = seq.Length;
        if (seqLen == 0) return start;
        if (maxLen < seqLen) return -1;

        byte first = seq[0];
        int limit = start + maxLen - seqLen + 1;

        for (int i = start; i < limit; i++)
        {
            if (data[i] != first) continue;
            bool match = true;
            for (int j = 1; j < seqLen; j++)
                if (data[i + j] != seq[j]) { match = false; break; }
            if (match) return i;
        }
        return -1;
    }

    struct AoBPattern
    {
        public byte[] Bytes;
        public bool[] Mask;
        public int Length;
        public int AnchorOffset;
        public byte[] AnchorSequence;
    }

    static AoBPattern ParseAoB(string sig)
    {
        if (string.IsNullOrWhiteSpace(sig)) return new AoBPattern { Length = 0 };
        var parts = sig.Split(new[] { ' ' }, StringSplitOptions.RemoveEmptyEntries);
        int n = parts.Length;
        var bytes = new byte[n];
        var mask = new bool[n];

        for (int i = 0; i < n; i++)
        {
            if (parts[i] == "??" || parts[i] == "?")
                mask[i] = false;
            else
            {
                mask[i] = true;
                bytes[i] = byte.Parse(parts[i], System.Globalization.NumberStyles.HexNumber);
            }
        }

        int bestStart = -1, bestLen = 0, curStart = -1, curLen = 0;
        for (int i = 0; i < n; i++)
        {
            if (mask[i])
            {
                if (curStart == -1) curStart = i;
                curLen++;
                if (curLen > bestLen) { bestLen = curLen; bestStart = curStart; }
            }
            else { curStart = -1; curLen = 0; }
        }

        byte[] anchor = bestStart >= 0 ? new byte[bestLen] : new byte[0];
        if (bestStart >= 0) Array.Copy(bytes, bestStart, anchor, 0, bestLen);

        return new AoBPattern
        {
            Bytes = bytes,
            Mask = mask,
            Length = n,
            AnchorOffset = bestStart,
            AnchorSequence = anchor
        };
    }
    #endregion

    #region Auto Pointer Chain Finder
    public class PointerChainResult
    {
        public long ModuleBase { get; set; }
        public int[] Offsets { get; set; }
        public long FinalAddress { get; set; }
        public override string ToString()
        {
            return string.Join(", ", Offsets.Select(o => "0x" + o.ToString("X")));
        }
    }

    class ChainNode
    {
        public long Address;
        public int OffsetFromParent;
        public ChainNode Parent;
    }

    public Task<PointerChainResult> AutoFindPointerChain(string aobSignature, string moduleName, int maxLevel, int maxOffset)
    {
        return Task.Run(() => AutoFindPointerChainSync(aobSignature, moduleName, maxLevel, maxOffset));
    }

    PointerChainResult AutoFindPointerChainSync(string aobSignature, string moduleName, int maxLevel, int maxOffset)
    {
        var targets = AoBScanSync(aobSignature, true, true);
        if (targets.Count == 0) return null;
        long target = targets[0];

        long modBase = GetModuleBase(moduleName);
        if (modBase == 0) return null;
        uint modSize = GetModuleSize(moduleName);
        if (modSize == 0) modSize = 0x2000000;

        var regions = GetRegions(false, false);
        var currentNodes = new List<ChainNode>
        {
            new ChainNode { Address = target, OffsetFromParent = 0, Parent = null }
        };

        const int MAX_LAYER_SIZE = 3000;

        for (int level = 0; level < maxLevel; level++)
        {
            var searchMap = new Dictionary<long, Tuple<ChainNode, int>>();
            foreach (var node in currentNodes)
            {
                for (int off = 0; off <= maxOffset; off += 4)
                {
                    long v = node.Address - off;
                    if (v < 0) continue;
                    if (!searchMap.ContainsKey(v))
                        searchMap[v] = Tuple.Create(node, off);
                }
            }

            var found = new ConcurrentBag<Tuple<long, long>>();
            Parallel.ForEach(regions, reg =>
            {
                ScanRegionForPointerValues(reg, new HashSet<long>(searchMap.Keys), found);
            });

            foreach (var tuple in found)
            {
                long ptrAddr = tuple.Item1;
                long value = tuple.Item2;

                if (ptrAddr >= modBase && ptrAddr < modBase + modSize)
                {
                    if (searchMap.ContainsKey(value))
                    {
                        var info = searchMap[value];
                        var offsets = new List<int>();
                        offsets.Add((int)(ptrAddr - modBase));
                        offsets.Add(info.Item2);

                        var node = info.Item1;
                        while (node != null)
                        {
                            if (node.Parent != null)
                                offsets.Add(node.OffsetFromParent);
                            node = node.Parent;
                        }

                        return new PointerChainResult
                        {
                            ModuleBase = modBase,
                            Offsets = offsets.ToArray(),
                            FinalAddress = target
                        };
                    }
                }
            }

            var nextNodes = new List<ChainNode>();
            foreach (var tuple in found)
            {
                long ptrAddr = tuple.Item1;
                long value = tuple.Item2;

                if (ptrAddr >= modBase && ptrAddr < modBase + modSize) continue;
                if (searchMap.ContainsKey(value))
                {
                    var info = searchMap[value];
                    nextNodes.Add(new ChainNode
                    {
                        Address = ptrAddr,
                        OffsetFromParent = info.Item2,
                        Parent = info.Item1
                    });
                }
            }

            currentNodes = nextNodes.GroupBy(n => n.Address)
                                    .Select(g => g.First())
                                    .Take(MAX_LAYER_SIZE)
                                    .ToList();

            if (currentNodes.Count == 0) break;
        }

        return null;
    }

    void ScanRegionForPointerValues(MemRegion reg, HashSet<long> searchValues, ConcurrentBag<Tuple<long, long>> results)
    {
        const int CHUNK = 4 * 1024 * 1024;
        byte[] buf = new byte[CHUNK];
        int pos = 0;

        while (pos < reg.Size)
        {
            int toRead = Math.Min(CHUNK, reg.Size - pos);
            IntPtr readAddr = IntPtr.Add(reg.Base, pos);
            if (!ReadProcessMemory(_handle, readAddr, buf, toRead, out int bytesRead) || bytesRead < 8)
            {
                pos += toRead;
                continue;
            }

            int limit = bytesRead - 7;
            for (int i = 0; i < limit; i += 8)
            {
                long val = BitConverter.ToInt64(buf, i);
                if (searchValues.Contains(val))
                    results.Add(Tuple.Create((long)readAddr + i, val));
            }

            pos += bytesRead;
        }
    }
    #endregion

    #region Read / Write
    public T ReadMemory<T>(string address) where T : struct { return ReadMemory<T>(ParseAddr(address)); }
    public T ReadMemory<T>(long address) where T : struct
    {
        int sz = Marshal.SizeOf<T>();
        byte[] buf = new byte[sz];

        if (!ReadProcessMemory(_handle, (IntPtr)address, buf, sz, out int rd) || rd != sz)
            throw new InvalidOperationException("Read failed at 0x" + address.ToString("X"));

        if (typeof(T) == typeof(int))   return (T)(object)BitConverter.ToInt32(buf, 0);
        if (typeof(T) == typeof(uint))  return (T)(object)BitConverter.ToUInt32(buf, 0);
        if (typeof(T) == typeof(long))  return (T)(object)BitConverter.ToInt64(buf, 0);
        if (typeof(T) == typeof(ulong)) return (T)(object)BitConverter.ToUInt64(buf, 0);
        if (typeof(T) == typeof(float)) return (T)(object)BitConverter.ToSingle(buf, 0);
        if (typeof(T) == typeof(double)) return (T)(object)BitConverter.ToDouble(buf, 0);
        if (typeof(T) == typeof(short)) return (T)(object)BitConverter.ToInt16(buf, 0);
        if (typeof(T) == typeof(ushort)) return (T)(object)BitConverter.ToUInt16(buf, 0);
        if (typeof(T) == typeof(byte))  return (T)(object)buf[0];
        if (typeof(T) == typeof(bool))  return (T)(object)BitConverter.ToBoolean(buf, 0);

        var h = GCHandle.Alloc(buf, GCHandleType.Pinned);
        try { return Marshal.PtrToStructure<T>(h.AddrOfPinnedObject()); }
        finally { h.Free(); }
    }

    public void WriteMemory(string address, string type, string value) { WriteMemory(ParseAddr(address), type, value); }
    public void WriteMemory(long address, string type, string value)
    {
        byte[] data;
        string t = type.ToLowerInvariant();

        if (t == "bytes" || t == "byte")
        {
            data = value.Split(new[] { ' ' }, StringSplitOptions.RemoveEmptyEntries)
                        .Select(b => Convert.ToByte(b, 16)).ToArray();
        }
        else if (t == "int")
            data = BitConverter.GetBytes(int.Parse(value));
        else if (t == "uint")
            data = BitConverter.GetBytes(uint.Parse(value));
        else if (t == "long")
            data = BitConverter.GetBytes(long.Parse(value));
        else if (t == "ulong")
            data = BitConverter.GetBytes(ulong.Parse(value));
        else if (t == "float")
            data = BitConverter.GetBytes(float.Parse(value));
        else if (t == "double")
            data = BitConverter.GetBytes(double.Parse(value));
        else if (t == "short")
            data = BitConverter.GetBytes(short.Parse(value));
        else if (t == "ushort")
            data = BitConverter.GetBytes(ushort.Parse(value));
        else if (t == "string")
            data = Encoding.UTF8.GetBytes(value + "\0");
        else
            throw new ArgumentException("Unknown type " + type);

        if (!WriteProcessMemory(_handle, (IntPtr)address, data, data.Length, out int wr) || wr != data.Length)
            throw new InvalidOperationException("Write failed at 0x" + address.ToString("X"));
    }

    static long ParseAddr(string a)
    {
        a = a.Trim();
        if (a.StartsWith("0x", StringComparison.OrdinalIgnoreCase)) a = a.Substring(2);
        return long.Parse(a, System.Globalization.NumberStyles.HexNumber);
    }
    #endregion

    public void Dispose()
    {
        if (_handle != IntPtr.Zero) { CloseHandle(_handle); _handle = IntPtr.Zero; }
    }
}
