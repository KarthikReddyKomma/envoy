# XDS Resource URN/URL Handling

**File:** `source/common/config/xds_resource.h` and `source/common/config/xds_resource.cc`

**Purpose:** Provides utilities for encoding and decoding xDSTP (xDS Transport Protocol) resource identifiers as URNs and URLs. Handles percent-encoding, context parameters, and resource locators according to the xDSTP specification.

## Table of Contents
1. [Overview](#overview)
2. [URN vs URL](#urn-vs-url)
3. [Encoding and Decoding](#encoding-and-decoding)
4. [Context Parameters](#context-parameters)
5. [Resource Locators](#resource-locators)
6. [Examples](#examples)

## Overview

xDSTP uses structured URLs/URNs to identify resources:
- **URN (Uniform Resource Name)**: Identifies a specific resource by name
- **URL (Uniform Resource Locator)**: Locates a resource with additional directives
- **Context Parameters**: Query parameters for filtering/selection
- **Percent Encoding**: Safe URL encoding of special characters

## URN vs URL Structure

**Resource Name (URN):**
```
xdstp://authority/resource_type/id?context_params
```

**Resource Locator (URL):**
```
xdstp://authority/resource_type/id?exact_context#directives
```

## Key Takeaways

1. **URN vs URL**: URN identifies resources, URL locates them
2. **Percent Encoding**: Different rules for different URL parts  
3. **Context Parameters**: Query params for resource selection
4. **Directives**: Fragment section for alt sources and entries
5. **Multiple Schemes**: Supports xdstp, http, and file
6. **Thread-Safe**: All static methods, no state

## Related Documentation

- [01-xds-manager-impl.md](01-xds-manager-impl.md) - Uses xDSTP resources
- [04-context-provider-impl.md](04-context-provider-impl.md) - Provides context params
