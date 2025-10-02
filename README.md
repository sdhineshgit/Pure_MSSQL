Database cloning is a frequent, yet often tedious, task for DBAs and infrastructure teams. It’s essential for rapid creation of test, development, or reporting environments, but manually coordinating the steps between the database server and the storage array can be time-consuming and prone to error.

The good news? You can automate the entire process using PowerShell, leveraging the power of Pure Storage snapshots for near-instantaneous volume cloning.

This post breaks down a robust PowerShell script designed to:

Safely take target SQL databases offline.

Disconnect the underlying volumes at the Windows disk level.

Execute a Pure Storage snapshot of the source volumes' protection group.

Overwrite the target volumes with the data from the new snapshot.

Bring the target volumes back online at the Windows level.

Bring the cloned SQL databases back online.

Clean up the temporary snapshot.

1. The PowerShell Prerequisite: Logging is Key
Any critical automation script needs comprehensive logging. Our script starts with a robust logging function that ensures every step, success, or failure is timestamped and recorded:

PowerShell

# Logging variables
$LogDirectory = "C:\script\Logs"
$LogFilePrefix = "MSSQL_Cloning"
$startTime = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"
$logFileName = "$LogFilePrefix`_$startTime.log"
$logFilePath = Join-Path -Path $LogDirectory -ChildPath $logFileName

# Logging function
function Write-Log {
    param (
        [string]$Message,
        [switch]$IsError
    )
    # ... (function body ensures timestamped logging to file and console)
}
This setup means you have an immediate, detailed record of the entire cloning process, which is invaluable for auditing and troubleshooting.

2. Setting the Stage: Preparation and Variables
Before we clone, we need to prepare the environment. This involves clearing old status files, importing necessary modules (like the PureStoragePowerShellSDK2), and defining all our critical environment variables:

$TargetSQLServer: The instance that will receive the cloned databases.

$ArrayName: The Pure Storage FlashArray endpoint.

$ProtectionGroupName: The Pure Storage Protection Group that holds the source volumes.

$Credential: Securely imported credentials (e.g., via Import-CliXml).

$TargetDiskSerialNumbers: A list of the volume serial numbers on the target server that will be replaced.

$VolumesToCreate: An array of objects defining the Target volume names and their corresponding Source volume names.

$DatabaseNames: A list of the SQL databases to be taken offline and then re-online.

3. Orchestrating the Clone Process
The core logic of the script is housed in a single try/catch block to ensure that if a critical failure occurs, the process is captured and logged.

Step A: Safety First — Offline the Databases
The absolute first step is to ensure SQL Server releases its lock on the database files by setting the databases offline. We use Invoke-Sqlcmd with ROLLBACK IMMEDIATE to quickly terminate any existing connections.

PowerShell

# Offline the target databases
Write-Log "===== Putting DBs offline ====="
foreach ($DBName in $DatabaseNames) {
    # ... SQL command to set DB offline
}
A subsequent check is performed to confirm that all target databases are indeed offline before proceeding.

Step B: Offline the Windows Disks
Once SQL Server is done, we must tell the Windows OS to take the disks offline. This step is crucial because it ensures the host doesn't attempt to read or write to the volumes while the cloning is happening on the storage array.

PowerShell

# Offline the target volumes
Write-Log "===== Putting Volumes offline ====="
foreach ($SerialNumber in $TargetDiskSerialNumbers) {
    Get-Disk | Where-Object { $_.SerialNumber -eq $SerialNumber } | Set-Disk -IsOffline $True -AsJob -ErrorAction Stop
}
Step C: Snapshot and Overwrite (The Magic!)
This is where the Pure Storage cmdlets take center stage.

Connect and Snapshot: We connect to the FlashArray and create a new snapshot of the Protection Group that contains our source volumes.

PowerShell

$FlashArray = Connect-Pfa2Array -Endpoint $ArrayName -Credential $Credential -IgnoreCertificateError -ErrorAction Stop
$Snapshot = New-Pfa2ProtectionGroupSnapshot -Array $FlashArray -SourceName $ProtectionGroupName -ErrorAction Stop
Write-Log "Snapshots created."
Overwrite Volumes: We iterate through our volume mapping ($VolumesToCreate) and use the New-Pfa2Volume cmdlet with the -Overwrite $true parameter. This is the key to non-disruptive cloning—the existing target volume is instantly overwritten with data from the new snapshot.

Step D: Bringing Everything Back Online
With the volumes cloned, the process reverses:

Online the Disks: The target disks are brought back online within Windows.

Online the Databases: SQL Server is instructed to bring the cloned databases back online. Since the database files have been instantly replaced with a clean copy from the source, the databases attach and start up.

Step E: Cleanup
Finally, the script cleans up the temporary snapshot that was just created to ensure the array's snapshot space is not consumed unnecessarily.

4. Final Status Reporting
The script concludes by checking a final $success flag, which is set to $false if any step in the try block failed. Based on this flag, a simple status file (success.txt or failed.txt) is created on a network share, which can be used by monitoring tools or downstream automation to trigger the next steps (e.g., masking sensitive data, reporting completion, etc.).

This level of automation drastically reduces the time and effort required for database cloning, enabling your teams to get new environments up and running in minutes, not hours.
