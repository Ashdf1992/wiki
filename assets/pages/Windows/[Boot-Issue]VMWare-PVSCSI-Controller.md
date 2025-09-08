# VMWare PVSCSI Controller Boot Issue – Resolution Guide

If you are experiencing boot issues after installing the **VMware, Inc. – SCSIAdapter 1.3.18.0** driver, follow the steps below based on your situation.

---

## 🟢 **If the Server Has NOT Been Rebooted**

If the driver is installed but the server has **not** been rebooted yet, you can roll back the driver to prevent boot issues:

**Steps:**
1. Open **Computer Management**.
2. Go to **Device Manager**.
3. Expand **Storage Controllers**.
4. Right-click **VMware PVSCSI Controller** and select **Properties**.
5. Go to the **Driver** tab.
6. Click **Rollback Driver**.

> After rolling back, you can safely reboot the server.

---

## 🔴 **If the Server HAS Been Rebooted (Stuck in Recovery)**

If the server is stuck in **Automatic/System Recovery** after rebooting, use these steps:

**Steps:**
1. During boot, press **F8** to access the boot menu.
2. Select **Disable Driver Signature Enforcement**.
   - Windows should now boot.
3. Once in Windows, repeat the rollback steps:
   - Open **Computer Management**.
   - Go to **Device Manager**.
   - Expand **Storage Controllers**.
   - Right-click **VMware PVSCSI Controller** and select **Properties**.
   - Go to the **Driver** tab.
   - Click **Rollback Driver**.

> After rolling back, you should be able to reboot without further issues.

---

### ℹ️ **Notes**
- Always confirm the driver version before making changes.
- If problems persist, contact VMware support.

---

**Tip:**  
Use the section relevant to your situation. If unsure, start with the "Not Rebooted" steps.
