# RDP Connection Failures: Expired Self-Signed Certificate

## Issue
Remote Desktop (RDP) connections begin to fail with no apparent cause.

## Symptoms
This issue might have the following symptoms:

- The client can't connect to the server by using RDP. Connection attempts return code 50331673: The Remote Desktop Gateway server administrator has ended the connection.
- The system logs register Event ID 36870 for every RPD connection attempt.

## Cause
The following events could cause this issue:

- The RDP self-signed certificate has expired or is missing (Windows® usually recreates the self-signed certificate upon expiration.
- Permissions issues on the following path: C:\ProgramData\Microsoft\Crypto\RSA\MachineKeys\f686aace6942fb7f7ceb231212eef4a4. The parent folder did not allow the OS to delete the existing key, which needs to happen before self-signed certificate recreation.

## Resolution
1) Delete the expired certificate from the Centralized Certificate Store (CCS) on the server by using the Certificates snap-in in the Microsoft Management Console (MMC). Select Certificates > Remote Desktop > Certificates.

2) Stop the RDP service.

3) Go to path C:\ProgramData\Microsoft\Crypto\RSA\MachineKeys, take ownership of the f686 key file, referenced previously, and give the owner of the file Full Control permission.

4) Change the Administrators group permission for the MachineKeys folder to apply to "This folder, subfolders and files.
   
5) Delete file: f686aace6942fb7f7ceb231212eef4a4.

6) Start the Remote Desktop Services service.

7) Verify that the system generated a new certificate by using the Certificates snap-in in MMC.

8) Verify RDP access to the server.
