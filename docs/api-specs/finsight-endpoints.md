# Zetheta FinSight Destination API Documentation

## Overview

This document provides comprehensive documentation for the destination endpoints used to load data into Zetheta FinSight analytics platform.

## Base URLs

| Environment | URL |
|-------------|-----|
| Production | `https://api.finsight.zetheta.com/v1` |
| QA | `https://api-qa.finsight.zetheta.com/v1` |
| Development | `https://api-dev.finsight.zetheta.com/v1` |

## Authentication

### OAuth 2.0 Client Credentials Flow

```http
POST https://auth.finsight.zetheta.com/oauth2/token
Content-Type: application/x-www-form-urlencoded

client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&grant_type=client_credentials
&scope=write:journal write:vendor write:customer write:costcentre