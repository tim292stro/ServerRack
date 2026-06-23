## Section 3. Diskless Provisioning Sequence

1. Open your desktop browser on Linux Mint and access **Server 1's IPMI control board** at `https://192.168.99.2`.
2. Navigate to **Virtual Media** → **CD-ROM Image / Virtual Storage**.
3. Choose **ISO File / Hard Disk Image** mode.
4. Input the share metrics:
   * **Share Host:** `192.168.99.1`
   * **Path to Image:** `\server-disks\server1-root.img`
   * **Username:** `kvmbootuser`
   * **Password:** *(Samba user password)*
5. Click **Mount / Plug-In**. Repeat this workflow on **Server 2's portal** (`https://192.168.99.6`), targeting host `192.168.99.5` and path `server2-root.img`.
6. Power on the hardware nodes, hit **F11**, and select the **ATEN Virtual Disk** profile as the primary boot target. Run a clean, headless **Debian 12 installation** targeted directly onto the virtual media volumes.
