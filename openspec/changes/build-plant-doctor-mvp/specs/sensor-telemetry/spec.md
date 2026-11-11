# Spec Delta

## Purpose

Collects the environmental evidence of the disease triangle (temperature, humidity, soil moisture, light, leaf wetness) from low-cost IoT nodes and makes it available to diagnosis and alerts.

## ADDED Requirements

### Requirement: Periodic telemetry ingestion
The system SHALL ingest temperature, relative humidity, soil moisture, light and leaf wetness from each sensor node every 5 minutes over MQTT and store them as a time series keyed by site and node.

#### Scenario: Normal reading
- **WHEN** a node publishes a valid JSON payload on `plantdoc/<site>/<node>/telemetry`
- **THEN** the reading is stored with its timestamp, site and node and is queryable within 1 minute

#### Scenario: Malformed payload
- **WHEN** a payload is missing required fields or has out-of-range values
- **THEN** the reading is rejected, logged, and does not enter the time series

### Requirement: Sensor window retrieval
The system SHALL return the sensor readings of a plant's node for a requested time window, and SHALL report data gaps explicitly.

#### Scenario: Window with gap
- **WHEN** diagnosis requests the last 72 h and the node was offline for 3 h
- **THEN** the window is returned with the 3 h gap marked, and weather data may be used to fill outdoor context

### Requirement: Leaf-wetness fallback
The system SHALL derive leaf-wetness hours from the physical sensor when available, and otherwise from the proxy RH ≥ 90%, and SHALL label which source was used.

#### Scenario: No wetness sensor
- **WHEN** a node has no leaf-wetness sensor
- **THEN** wetness hours are computed from hours with RH ≥ 90% and labelled as "proxy"

### Requirement: Low-cost node
Each sensor node's bill of materials SHALL cost NT$1,500 or less.

#### Scenario: BOM review
- **WHEN** the node BOM is priced
- **THEN** the total is ≤ NT$1,500
