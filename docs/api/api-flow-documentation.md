# API Flow Documentation: Postcode to Stock Check

## Overview

This document outlines the complete API flow for checking product stock at nearby stores using a customer's postcode. The process involves four sequential steps that transform a simple postcode input into actionable stock availability information.

## Flow Sequence

```
Postcode → Coordinates → Nearby Stores → Stock Check
```

## Step 1: Postcode to Coordinates

### Purpose
Convert a user-provided postcode into geographical coordinates (latitude and longitude) for location-based services.

### API Endpoint
```
GET /api/geocoding/postcode/{postcode}
```

### Request Example
```http
GET /api/geocoding/postcode/SW1A1AA
```

### Request Parameters
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| postcode | string | Yes | Valid UK postcode format |

### Response Format
```json
{
  "success": true,
  "data": {
    "postcode": "SW1A 1AA",
    "latitude": 51.501364,
    "longitude": -0.141890,
    "accuracy": "high",
    "region": "London",
    "country": "UK"
  },
  "timestamp": "2025-10-16T11:59:00Z"
}
```

### Error Handling
```json
{
  "success": false,
  "error": {
    "code": "INVALID_POSTCODE",
    "message": "The provided postcode format is invalid",
    "details": "Expected format: AA1A 1AA or AA1 1AA"
  }
}
```

### Validation Rules
- Postcode must match UK format patterns
- Handle both spaced and unspaced formats
- Case insensitive input
- Return standardized format in response

## Step 2: Coordinates to Nearby Stores

### Purpose
Find all store locations within a specified radius of the given coordinates.

### API Endpoint
```
GET /api/stores/nearby?lat={latitude}&lng={longitude}&radius={radius}
```

### Request Example
```http
GET /api/stores/nearby?lat=51.501364&lng=-0.141890&radius=5000
```

### Request Parameters
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| lat | decimal | Yes | - | Latitude coordinate |
| lng | decimal | Yes | - | Longitude coordinate |
| radius | integer | No | 5000 | Search radius in meters |
| limit | integer | No | 20 | Maximum stores to return |
| store_type | string | No | all | Filter by store type |

### Response Format
```json
{
  "success": true,
  "data": {
    "total_found": 8,
    "search_radius": 5000,
    "center_point": {
      "latitude": 51.501364,
      "longitude": -0.141890
    },
    "stores": [
      {
        "store_id": "ST001",
        "name": "Central London Store",
        "address": {
          "street": "123 High Street",
          "city": "London",
          "postcode": "SW1A 2BB"
        },
        "coordinates": {
          "latitude": 51.502364,
          "longitude": -0.140890
        },
        "distance_meters": 150,
        "store_type": "flagship",
        "opening_hours": {
          "monday": "09:00-18:00",
          "tuesday": "09:00-18:00",
          "wednesday": "09:00-18:00",
          "thursday": "09:00-20:00",
          "friday": "09:00-20:00",
          "saturday": "09:00-17:00",
          "sunday": "11:00-16:00"
        },
        "contact": {
          "phone": "+44 20 1234 5678",
          "email": "central@example.com"
        }
      }
    ]
  },
  "timestamp": "2025-10-16T11:59:01Z"
}
```

### Sorting Options
- **distance** (default): Closest stores first
- **name**: Alphabetical order
- **store_type**: Flagship stores first

## Step 3: Stock Check at Multiple Stores

### Purpose
Check product availability and stock levels at the identified nearby stores.

### API Endpoint
```
POST /api/inventory/bulk-check
```

### Request Example
```http
POST /api/inventory/bulk-check
Content-Type: application/json

{
  "product_id": "PRD123456",
  "store_ids": ["ST001", "ST002", "ST003"],
  "include_alternatives": true
}
```

### Request Body Parameters
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| product_id | string | Yes | Unique product identifier |
| store_ids | array | Yes | List of store IDs to check |
| include_alternatives | boolean | No | Include similar products |
| size | string | No | Specific size variant |
| color | string | No | Specific color variant |

### Response Format
```json
{
  "success": true,
  "data": {
    "product": {
      "id": "PRD123456",
      "name": "Classic T-Shirt",
      "sku": "TSH-CLX-001",
      "price": {
        "current": 24.99,
        "currency": "GBP"
      }
    },
    "stock_results": [
      {
        "store_id": "ST001",
        "store_name": "Central London Store",
        "availability": "in_stock",
        "quantity": 15,
        "variants": [
          {
            "size": "M",
            "color": "Blue",
            "quantity": 5
          },
          {
            "size": "L",
            "color": "Blue",
            "quantity": 3
          },
          {
            "size": "M",
            "color": "Red",
            "quantity": 7
          }
        ],
        "last_updated": "2025-10-16T11:45:00Z",
        "reservation_expires": "2025-10-16T12:59:00Z"
      },
      {
        "store_id": "ST002",
        "store_name": "West End Store",
        "availability": "low_stock",
        "quantity": 2,
        "variants": [
          {
            "size": "S",
            "color": "Blue",
            "quantity": 2
          }
        ],
        "last_updated": "2025-10-16T11:30:00Z"
      },
      {
        "store_id": "ST003",
        "store_name": "Camden Store",
        "availability": "out_of_stock",
        "quantity": 0,
        "estimated_restock": "2025-10-18T09:00:00Z"
      }
    ],
    "alternatives": [
      {
        "product_id": "PRD123457",
        "name": "Premium T-Shirt",
        "similarity_score": 0.85,
        "available_stores": ["ST001", "ST002"]
      }
    ]
  },
  "timestamp": "2025-10-16T11:59:02Z"
}
```

### Stock Status Definitions
- **in_stock**: Available quantity > 5
- **low_stock**: Available quantity 1-5
- **out_of_stock**: Available quantity = 0
- **discontinued**: Product no longer carried

## Complete Flow Implementation

### Sequential Processing Example

```javascript
async function checkProductAvailability(postcode, productId) {
  try {
    // Step 1: Get coordinates from postcode
    const coordinates = await geocodePostcode(postcode);
    
    // Step 2: Find nearby stores
    const nearbyStores = await findNearbyStores(
      coordinates.latitude, 
      coordinates.longitude
    );
    
    // Step 3: Check stock at all nearby stores
    const storeIds = nearbyStores.stores.map(store => store.store_id);
    const stockResults = await checkStockAtStores(productId, storeIds);
    
    return {
      location: coordinates,
      stores: nearbyStores.stores,
      stock: stockResults
    };
    
  } catch (error) {
    console.error('Flow failed:', error);
    throw error;
  }
}
```

### Error Handling Strategy

1. **Graceful Degradation**: If one step fails, provide partial results
2. **Retry Logic**: Implement exponential backoff for transient failures
3. **Fallback Options**: Use cached data when services are unavailable
4. **User Communication**: Provide clear error messages and next steps

### Rate Limiting

- **Geocoding**: 1000 requests/hour per API key
- **Store Lookup**: 5000 requests/hour per API key  
- **Stock Check**: 10000 requests/hour per API key

### Caching Recommendations

- **Coordinates**: Cache for 24 hours (postcodes rarely change)
- **Store Locations**: Cache for 1 hour (store hours may change)
- **Stock Data**: Cache for 5 minutes maximum (high volatility)

## Testing Scenarios

### Happy Path
1. Valid postcode → coordinates found
2. Coordinates → multiple stores found within radius
3. Stores → stock available at some locations

### Edge Cases
1. **Invalid postcode**: Return clear error message
2. **No nearby stores**: Expand search radius automatically
3. **Out of stock everywhere**: Suggest alternatives or restock dates
4. **Service unavailable**: Use cached data where possible

### Performance Considerations

- **Parallel Processing**: Execute store stock checks concurrently
- **Batch Requests**: Group multiple product checks when possible
- **Geographic Optimization**: Pre-filter stores by rough distance
- **Response Compression**: Use gzip for large store lists

## Security & Privacy

### Data Protection
- Never log or store customer postcodes
- Implement request rate limiting
- Use HTTPS for all API communications
- Sanitize all input parameters

### Authentication
```http
Authorization: Bearer {jwt_token}
X-API-Key: {api_key}
```

## Monitoring & Analytics

### Key Metrics
- **Conversion Rate**: Postcode → successful stock check
- **Average Response Time**: Per API endpoint
- **Error Rates**: By step and error type
- **Geographic Coverage**: Areas with poor store density

### Logging Structure
```json
{
  "timestamp": "2025-10-16T11:59:03Z",
  "flow_id": "FL-123456789",
  "step": "stock_check",
  "duration_ms": 245,
  "success": true,
  "store_count": 3,
  "product_id": "PRD123456"
}
```

## Conclusion

This API flow provides a seamless customer experience by transforming a simple postcode into actionable product availability information. The sequential nature ensures data accuracy while the error handling and caching strategies maintain performance and reliability.

For implementation questions or API access, contact the development team or refer to the complete API specification document.