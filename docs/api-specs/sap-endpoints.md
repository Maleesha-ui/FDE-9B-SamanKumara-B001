# ============================================
# SRC-001: General Ledger Journal Entries
# ============================================
paths:
  /gl/journal-entries:
    get:
      tags:
        - General Ledger
      summary: Get GL Journal Entries
      description: |
        Extracts General Ledger journal entries from SAP Universal Journal (ACDOCA).
        Supports delta extraction via ODP framework.
      operationId: getGLJournalEntries
      security:
        - OAuth2:
            - read:gl
        - RFCConnection: []
      parameters:
        - $ref: '#/components/parameters/DeltaToken'
        - $ref: '#/components/parameters/CompanyCode'
        - $ref: '#/components/parameters/FiscalYear'
        - $ref: '#/components/parameters/PostingPeriod'
        - $ref: '#/components/parameters/Limit'
        - $ref: '#/components/parameters/Offset'
        - name: documentType
          in: query
          description: Document Type filter
          schema:
            type: string
            enum: [SA, RV, KR, KA, WA, AB, DZ, KZ]
        - name: glAccount
          in: query
          description: G/L Account filter
          schema:
            type: string
            pattern: ^[0-9]{6}$
        - name: fromDate
          in: query
          description: From posting date
          schema:
            type: string
            format: date
        - name: toDate
          in: query
          description: To posting date
          schema:
            type: string
            format: date
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/JournalEntry'
                  pagination:
                    type: object
                    properties:
                      total:
                        type: integer
                        description: Total records
                      limit:
                        type: integer
                        description: Limit parameter
                      offset:
                        type: integer
                        description: Offset parameter
                  deltaToken:
                    type: string
                    description: Next delta token for incremental extraction
                    example: "20260328_000002_001"
        '400':
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '401':
          description: Unauthorized
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '429':
          description: Rate Limited
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '500':
          description: Internal Server Error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

# ============================================
# SRC-002: Accounts Payable - Vendor Master
# ============================================
  /ap/vendors:
    get:
      tags:
        - Accounts Payable
      summary: Get Vendor Master Data
      description: Extracts vendor master data from SAP (LFA1, LFC1)
      operationId: getVendors
      security:
        - OAuth2:
            - read:ap
        - RFCConnection: []
      parameters:
        - $ref: '#/components/parameters/CompanyCode'
        - $ref: '#/components/parameters/Limit'
        - $ref: '#/components/parameters/Offset'
        - name: vendorId
          in: query
          description: Vendor ID filter
          schema:
            type: string
        - name: gstNumber
          in: query
          description: GST Number filter
          schema:
            type: string
            pattern: ^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z]{1}[1-9A-Z]{1}[Z]{1}[0-9A-Z]{1}$
        - name: includeOpenItems
          in: query
          description: Include open items
          schema:
            type: boolean
            default: false
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Vendor'
                  pagination:
                    type: object
                    properties:
                      total:
                        type: integer
                      limit:
                        type: integer
                      offset:
                        type: integer

# ============================================
# SRC-003: Accounts Payable - Open Items
# ============================================
  /ap/open-items:
    get:
      tags:
        - Accounts Payable
      summary: Get AP Open Items
      description: Extracts vendor open items from SAP (BSIK)
      operationId: getAPOpenItems
      security:
        - OAuth2:
            - read:ap
        - RFCConnection: []
      parameters:
        - $ref: '#/components/parameters/CompanyCode'
        - name: vendorId
          in: query
          description: Vendor ID filter
          schema:
            type: string
        - name: agingDate
          in: query
          description: Aging date for calculation
          schema:
            type: string
            format: date
            default: current date
        - name: includeCleared
          in: query
          description: Include cleared items
          schema:
            type: boolean
            default: false
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      type: object
                      properties:
                        vendorId:
                          type: string
                        documentNumber:
                          type: string
                        documentType:
                          type: string
                        postingDate:
                          type: string
                          format: date
                        dueDate:
                          type: string
                          format: date
                        amount:
                          type: number
                          format: double
                        currency:
                          type: string
                          pattern: ^[A-Z]{3}$
                        agingBucket:
                          type: string
                          enum: [0-30, 31-60, 61-90, 91-120, 120+]
                        clearingStatus:
                          type: string
                          enum: [OPEN, CLEARED]
                  pagination:
                    type: object
                    properties:
                      total:
                        type: integer
                      limit:
                        type: integer
                      offset:
                        type: integer

# ============================================
# SRC-004: Accounts Receivable - Customer Master
# ============================================
  /ar/customers:
    get:
      tags:
        - Accounts Receivable
      summary: Get Customer Master Data
      description: Extracts customer master data from SAP (KNA1, KNC1)
      operationId: getCustomers
      security:
        - OAuth2:
            - read:ar
        - RFCConnection: []
      parameters:
        - $ref: '#/components/parameters/CompanyCode'
        - $ref: '#/components/parameters/Limit'
        - $ref: '#/components/parameters/Offset'
        - name: customerId
          in: query
          description: Customer ID filter
          schema:
            type: string
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Customer'
                  pagination:
                    type: object
                    properties:
                      total:
                        type: integer
                      limit:
                        type: integer
                      offset:
                        type: integer

# ============================================
# SRC-005: Accounts Receivable - Open Items
# ============================================
  /ar/open-items:
    get:
      tags:
        - Accounts Receivable
      summary: Get AR Open Items
      description: Extracts customer open items from SAP (BSID)
      operationId: getAROpenItems
      security:
        - OAuth2:
            - read:ar
        - RFCConnection: []
      parameters:
        - $ref: '#/components/parameters/CompanyCode'
        - name: customerId
          in: query
          description: Customer ID filter
          schema:
            type: string
        - name: agingDate
          in: query
          description: Aging date for calculation
          schema:
            type: string
            format: date
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      type: object
                      properties:
                        customerId:
                          type: string
                        documentNumber:
                          type: string
                        postingDate:
                          type: string
                          format: date
                        dueDate:
                          type: string
                          format: date
                        amount:
                          type: number
                          format: double
                        currency:
                          type: string
                          pattern: ^[A-Z]{3}$
                        agingBucket:
                          type: string
                          enum: [0-30, 31-60, 61-90, 91-120, 120+]
                        dunningLevel:
                          type: integer
                          description: Dunning level
                        creditExposure:
                          type: number
                          format: double
                        availableCredit:
                          type: number
                          format: double

# ============================================
# SRC-006: Cost Centres
# ============================================
  /cost-centres:
    get:
      tags:
        - Cost Accounting
      summary: Get Cost Centre Master Data
      description: Extracts cost centre master data from SAP (CSKS, CSKT)
      operationId: getCostCentres
      security:
        - OAuth2:
            - read:cost
        - RFCConnection: []
      parameters:
        - $ref: '#/components/parameters/CompanyCode'
        - $ref: '#/components/parameters/Limit'
        - $ref: '#/components/parameters/Offset'
        - name: costCentreId
          in: query
          description: Cost Centre ID filter
          schema:
            type: string
        - name: includeHierarchy
          in: query
          description: Include full hierarchy
          schema:
            type: boolean
            default: true
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/CostCentre'
                  pagination:
                    type: object
                    properties:
                      total:
                        type: integer
                      limit:
                        type: integer
                      offset:
                        type: integer