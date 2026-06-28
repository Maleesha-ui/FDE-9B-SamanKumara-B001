# Requirements Document - Meridian Manufacturing Integration

## 1. Technology Landscape

### Source Systems (On-Premise)

| System           | Version               | Role                              | Data Volume             |
| ---------------- | --------------------- | --------------------------------- | ----------------------- |
| SAP S/4HANA      | 2023 FPS02            | Core financial & operational data | 2.1M transactions/month |
| Salesforce       | Enterprise Winter '24 | Customer orders & pipeline        | 45K records/month       |
| Siemens Opcenter | 2023.1                | Shop floor production data        | 8.5M events/month       |
| Azure AD         | Latest                | Identity & authentication         | 3,200 user accounts     |

### Target Systems (Cloud - AWS Mumbai)

| System           | Version    | Role                            |
| ---------------- | ---------- | ------------------------------- |
| Zetheta FinSight | 4.2        | Financial analytics & reporting |
| Snowflake        | Enterprise | Historical data storage (15TB)  |
| Nagios + Grafana | N/A        | Infrastructure monitoring       |

### Network

* MPLS + SD-WAN between Pune DC and AWS Mumbai
* Average throughput: 450 Mbps
* Shared link - integration must not consume >25% during business hours

## 2. Plant Locations

| Code | Location       | SAP Company Code | Products                 | Monthly Transactions |
| ---- | -------------- | ---------------- | ------------------------ | -------------------- |
| PL01 | Pune, MH       | MC01             | Engine components        | 480,000              |
| PL02 | Nashik, MH     | MC01             | Transmission parts       | 320,000              |
| PL03 | Aurangabad, MH | MC01             | Precision bearings       | 280,000              |
| PL04 | Ahmedabad, GJ  | MC02             | Aerospace fasteners      | 190,000              |
| PL05 | Rajkot, GJ     | MC02             | Industrial valves        | 220,000              |
| PL06 | Chennai, TN    | MC03             | Electronic control units | 340,000              |
| PL07 | Coimbatore, TN | MC03             | Hydraulic systems        | 270,000              |

## 3. Business Constraints

1. **SAP Batch Window:** 01:00-04:30 IST - No heavy extraction allowed
2. **SAP RFC Pool:** Max 50 concurrent connections
3. **ODP Frequency:** Max once every 30 minutes per provider
4. **Network Bandwidth:** Max 25% (112.5 Mbps) during business hours
5. **Maintenance Windows:**

   * SAP: 2nd & 4th Saturday 22:00-06:00 IST
   * FinSight: 1st Sunday 02:00-06:00 IST
6. **Data Residency:** All data must stay within Indian borders (RBI mandate)
7. **Audit:** Complete data lineage required
8. **GST Compliance:** Preserve CGST/SGST/IGST breakdown

## 4. Success Criteria

1. Data available in FinSight within 4 hours of posting in SAP
2. Zero data loss during failures
3. Reconciliation breaks < 0.01% of transactions
4. Audit trail from SAP document to FinSight record
5. No negative impact on SAP production performance
