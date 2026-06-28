# Fiscal Period Mapping Logic

## Overview

SAP S/4HANA uses fiscal year variant V3 (April-March) which is standard for Indian companies. This document defines the mapping logic for converting SAP posting periods to calendar months and fiscal periods for FinSight analytics.

## Period Structure

### SAP Posting Periods
| Period | Description | Type |
|--------|-------------|------|
| 001 | April | Regular |
| 002 | May | Regular |
| 003 | June | Regular |
| 004 | July | Regular |
| 005 | August | Regular |
| 006 | September | Regular |
| 007 | October | Regular |
| 008 | November | Regular |
| 009 | December | Regular |
| 010 | January | Regular |
| 011 | February | Regular |
| 012 | March | Regular |
| 013 | March (Year-end adjustment 1) | Special |
| 014 | March (Year-end adjustment 2) | Special |
| 015 | March (Year-end adjustment 3) | Special |
| 016 | March (Year-end adjustment 4) | Special |

---

## Period Mapping Table

### SAP Period → Calendar Month

| SAP Period | Calendar Month | Month Number | Quarter |
|------------|----------------|--------------|---------|
| 001 | April | 4 | Q1 |
| 002 | May | 5 | Q1 |
| 003 | June | 6 | Q1 |
| 004 | July | 7 | Q2 |
| 005 | August | 8 | Q2 |
| 006 | September | 9 | Q2 |
| 007 | October | 10 | Q3 |
| 008 | November | 11 | Q3 |
| 009 | December | 12 | Q3 |
| 010 | January | 1 | Q4 |
| 011 | February | 2 | Q4 |
| 012 | March | 3 | Q4 |
| 013 | March | 3 | Q4 |
| 014 | March | 3 | Q4 |
| 015 | March | 3 | Q4 |
| 016 | March | 3 | Q4 |

### SAP Period → Fiscal Month

| SAP Period | Fiscal Month | Fiscal Quarter |
|------------|--------------|----------------|
| 001 | 1 | Q1 |
| 002 | 2 | Q1 |
| 003 | 3 | Q1 |
| 004 | 4 | Q2 |
| 005 | 5 | Q2 |
| 006 | 6 | Q2 |
| 007 | 7 | Q3 |
| 008 | 8 | Q3 |
| 009 | 9 | Q3 |
| 010 | 10 | Q4 |
| 011 | 11 | Q4 |
| 012 | 12 | Q4 |
| 013 | 12 | Q4 |
| 014 | 12 | Q4 |
| 015 | 12 | Q4 |
| 016 | 12 | Q4 |

---

## Transformation Logic

### 1. Basic Period Mapping

```python
def map_sap_period(period: str) -> dict:
    """
    Map SAP posting period to calendar month and fiscal period.
    
    Args:
        period (str): SAP period (001-016)
    
    Returns:
        dict: {
            'period': str,
            'calendar_month': str,
            'calendar_month_number': int,
            'fiscal_month': int,
            'quarter': str,
            'is_special': bool
        }
    """
    
    period_int = int(period)
    
    # Define period mappings
    period_mapping = {
        1: {'month': 'April', 'fiscal_month': 1, 'quarter': 'Q1'},
        2: {'month': 'May', 'fiscal_month': 2, 'quarter': 'Q1'},
        3: {'month': 'June', 'fiscal_month': 3, 'quarter': 'Q1'},
        4: {'month': 'July', 'fiscal_month': 4, 'quarter': 'Q2'},
        5: {'month': 'August', 'fiscal_month': 5, 'quarter': 'Q2'},
        6: {'month': 'September', 'fiscal_month': 6, 'quarter': 'Q2'},
        7: {'month': 'October', 'fiscal_month': 7, 'quarter': 'Q3'},
        8: {'month': 'November', 'fiscal_month': 8, 'quarter': 'Q3'},
        9: {'month': 'December', 'fiscal_month': 9, 'quarter': 'Q3'},
        10: {'month': 'January', 'fiscal_month': 10, 'quarter': 'Q4'},
        11: {'month': 'February', 'fiscal_month': 11, 'quarter': 'Q4'},
        12: {'month': 'March', 'fiscal_month': 12, 'quarter': 'Q4'},
    }
    
    # Special periods (013-016) map to March
    if 13 <= period_int <= 16:
        return {
            'period': period,
            'calendar_month': 'March',
            'calendar_month_number': 3,
            'fiscal_month': 12,
            'quarter': 'Q4',
            'is_special': True
        }
    
    # Regular period (001-012)
    mapping = period_mapping.get(period_int)
    if mapping:
        return {
            'period': period,
            'calendar_month': mapping['month'],
            'calendar_month_number': get_month_number(mapping['month']),
            'fiscal_month': mapping['fiscal_month'],
            'quarter': mapping['quarter'],
            'is_special': False
        }
    
    # Invalid period
    raise ValueError(f"Invalid SAP period: {period}")

def get_month_number(month_name: str) -> int:
    """Convert month name to number."""
    month_map = {
        'January': 1, 'February': 2, 'March': 3, 'April': 4,
        'May': 5, 'June': 6, 'July': 7, 'August': 8,
        'September': 9, 'October': 10, 'November': 11, 'December': 12
    }
    return month_map.get(month_name, 0)

    def get_fiscal_period_from_date(date: str) -> dict:
    """
    Determine fiscal period from a date.
    
    Args:
        date (str): Date in YYYY-MM-DD format
    
    Returns:
        dict: {
            'fiscal_year': str,
            'fiscal_period': str,
            'calendar_month': str,
            'quarter': str
        }
    """
    
    from datetime import datetime
    
    # Parse date
    dt = datetime.strptime(date, '%Y-%m-%d')
    
    # Indian fiscal year runs April to March
    if dt.month >= 4:
        fiscal_year = str(dt.year)
        fiscal_month = dt.month - 3
    else:
        fiscal_year = str(dt.year - 1)
        fiscal_month = dt.month + 9
    
    # Determine period
    period = f"{fiscal_month:03d}"
    
    # Get calendar month
    calendar_month = dt.strftime('%B')
    
    # Determine quarter
    if 1 <= fiscal_month <= 3:
        quarter = 'Q1'
    elif 4 <= fiscal_month <= 6:
        quarter = 'Q2'
    elif 7 <= fiscal_month <= 9:
        quarter = 'Q3'
    else:
        quarter = 'Q4'
    
    return {
        'fiscal_year': fiscal_year,
        'fiscal_period': period,
        'calendar_month': calendar_month,
        'quarter': quarter
    }

    -- SQL mapping for period conversion
SELECT 
    MONAT AS sap_period,
    CASE 
        -- Regular periods
        WHEN MONAT = '001' THEN 'April'
        WHEN MONAT = '002' THEN 'May'
        WHEN MONAT = '003' THEN 'June'
        WHEN MONAT = '004' THEN 'July'
        WHEN MONAT = '005' THEN 'August'
        WHEN MONAT = '006' THEN 'September'
        WHEN MONAT = '007' THEN 'October'
        WHEN MONAT = '008' THEN 'November'
        WHEN MONAT = '009' THEN 'December'
        WHEN MONAT = '010' THEN 'January'
        WHEN MONAT = '011' THEN 'February'
        WHEN MONAT = '012' THEN 'March'
        -- Special periods (013-016)
        WHEN MONAT IN ('013', '014', '015', '016') THEN 'March'
        ELSE 'Unknown'
    END AS calendar_month,
    CASE 
        WHEN MONAT IN ('001', '002', '003') THEN 'Q1'
        WHEN MONAT IN ('004', '005', '006') THEN 'Q2'
        WHEN MONAT IN ('007', '008', '009') THEN 'Q3'
        WHEN MONAT IN ('010', '011', '012', '013', '014', '015', '016') THEN 'Q4'
        ELSE 'Unknown'
    END AS fiscal_quarter,
    CASE 
        WHEN MONAT IN ('013', '014', '015', '016') THEN TRUE
        ELSE FALSE
    END AS is_special_period,
    CASE 
        WHEN MONAT = '001' THEN 1
        WHEN MONAT = '002' THEN 2
        WHEN MONAT = '003' THEN 3
        WHEN MONAT = '004' THEN 4
        WHEN MONAT = '005' THEN 5
        WHEN MONAT = '006' THEN 6
        WHEN MONAT = '007' THEN 7
        WHEN MONAT = '008' THEN 8
        WHEN MONAT = '009' THEN 9
        WHEN MONAT = '010' THEN 10
        WHEN MONAT = '011' THEN 11
        WHEN MONAT = '012' THEN 12
        WHEN MONAT IN ('013', '014', '015', '016') THEN 12
        ELSE 0
    END AS fiscal_month
FROM ACDOCA

-- models/staging/stg_fiscal_period.sql

WITH periods AS (
    SELECT 
        *,
        -- Extract period
        MONAT AS sap_period,
        GJAHR AS fiscal_year,
        
        -- Map to calendar month
        CASE 
            WHEN MONAT = '001' THEN 'April'
            WHEN MONAT = '002' THEN 'May'
            WHEN MONAT = '003' THEN 'June'
            WHEN MONAT = '004' THEN 'July'
            WHEN MONAT = '005' THEN 'August'
            WHEN MONAT = '006' THEN 'September'
            WHEN MONAT = '007' THEN 'October'
            WHEN MONAT = '008' THEN 'November'
            WHEN MONAT = '009' THEN 'December'
            WHEN MONAT = '010' THEN 'January'
            WHEN MONAT = '011' THEN 'February'
            WHEN MONAT = '012' THEN 'March'
            WHEN MONAT IN ('013', '014', '015', '016') THEN 'March'
        END AS calendar_month,
        
        -- Determine quarter
        CASE 
            WHEN MONAT IN ('001', '002', '003') THEN 1
            WHEN MONAT IN ('004', '005', '006') THEN 2
            WHEN MONAT IN ('007', '008', '009') THEN 3
            WHEN MONAT IN ('010', '011', '012', '013', '014', '015', '016') THEN 4
        END AS fiscal_quarter,
        
        -- Special period flag
        CASE 
            WHEN MONAT IN ('013', '014', '015', '016') THEN TRUE
            ELSE FALSE
        END AS is_special_period,
        
        -- Fiscal month number
        CASE 
            WHEN MONAT = '001' THEN 1
            WHEN MONAT = '002' THEN 2
            WHEN MONAT = '003' THEN 3
            WHEN MONAT = '004' THEN 4
            WHEN MONAT = '005' THEN 5
            WHEN MONAT = '006' THEN 6
            WHEN MONAT = '007' THEN 7
            WHEN MONAT = '008' THEN 8
            WHEN MONAT = '009' THEN 9
            WHEN MONAT = '010' THEN 10
            WHEN MONAT = '011' THEN 11
            WHEN MONAT = '012' THEN 12
            WHEN MONAT IN ('013', '014', '015', '016') THEN 12
        END AS fiscal_month
        
    FROM {{ source('sap', 'acdoca') }}
)

SELECT * FROM periods

Input: period = '003', fiscal_year = '2026'
Output: {
    'calendar_month': 'June',
    'fiscal_month': 3,
    'quarter': 'Q1',
    'is_special': False,
    'fiscal_year': '2026'
}

Input: period = '014', fiscal_year = '2026'
Output: {
    'calendar_month': 'March',
    'fiscal_month': 12,
    'quarter': 'Q4',
    'is_special': True,
    'fiscal_year': '2026'
}
Input: date = '2026-03-28'
Output: {
    'fiscal_year': '2026',
    'fiscal_period': '012',
    'calendar_month': 'March',
    'quarter': 'Q4'
}
Input: date = '2026-04-15'
Output: {
    'fiscal_year': '2026',
    'fiscal_period': '001',
    'calendar_month': 'April',
    'quarter': 'Q1'
}
{
    "fiscalYear": "2026",
    "postingPeriod": "003",
    "calendarMonth": "June",
    "fiscalMonth": 3,
    "quarter": "Q1",
    "isSpecialPeriod": false,
    "periodStartDate": "2026-06-01",
    "periodEndDate": "2026-06-30"
}
def get_period_dates(fiscal_year: str, period: str) -> dict:
    """
    Calculate period start and end dates.
    
    Args:
        fiscal_year (str): Fiscal year (e.g., '2026')
        period (str): SAP period (e.g., '003')
    
    Returns:
        dict: {
            'start_date': str,
            'end_date': str,
            'days_in_period': int
        }
    """
    
    period_int = int(period)
    year = int(fiscal_year)
    
    # Period 001 = April of fiscal year
    month = period_int + 3
    
    if month > 12:
        month = month - 12
        year = year + 1
    
    # Calculate start date
    start_date = f"{year}-{month:02d}-01"
    
    # Calculate end date
    if month == 12:
        end_date = f"{year}-12-31"
    else:
        end_date = f"{year}-{month + 1:02d}-01"
    
    return {
        'start_date': start_date,
        'end_date': end_date,
        'days_in_period': (datetime.strptime(end_date) - datetime.strptime(start_date)).days
    }