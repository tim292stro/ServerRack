## Section 10. Operational Status Check

To verify the serial tracking chain and check the status of the atomic physics package inside the PRS10, run the tracking metrics via the SatPulse diagnostic monitor on your Linux Mint PC:

```bash
satpulse-cli monitor
```

The terminal output will display real-time physical telemetry pulled directly via the serial connections:
*   **GNSS Fix:** `3D/DGNSS / Satellites Tracked: 24+`
*   **PRS10 Lock Status:** `Locked to Rubidium Resonance`
*   **Lamp Voltage:** `~4.1V` (Verifies atomic lamp lifetime health)
*   **System Time Drift:** `±0.000000002 seconds` (Sub-microsecond synchronization achieved)

Test your end-to-end certificate acquisition process manually using the ACME staging sandbox to verify integration before launching live traffic:
```bash
sudo certbot renew --dry-run
```
