# Spec Delta

## Purpose

Warns growers before symptoms appear by tracking weather-driven disease risk (TOMCAST disease-severity values and a gray-mold index) from sensor data.

## ADDED Requirements

### Requirement: TOMCAST DSV accumulation
The system SHALL compute a daily TOMCAST disease-severity value (DSV) per plant site from leaf-wetness hours and mean temperature, and SHALL accumulate it since the last reset.

#### Scenario: Daily update
- **WHEN** a day of sensor data is complete
- **THEN** that day's DSV is added to the site's accumulated DSV

### Requirement: Proactive DSV alert
The system SHALL send a proactive alert when accumulated DSV reaches the threshold (initially 15), and SHALL reset the accumulator when the user confirms a protective action.

#### Scenario: Threshold reached
- **WHEN** the accumulated DSV reaches 15
- **THEN** the user receives an alert with the DSV trend and cultural precautions

#### Scenario: User confirms action
- **WHEN** the user confirms a protective action after an alert
- **THEN** the accumulated DSV is reset to 0

### Requirement: Gray-mold risk index
The system SHALL compute a gray-mold risk index as the share of hours at 15–25 °C with RH ≥ 90% over the recent window and report it with the alert or diagnosis.

#### Scenario: Humid cool week
- **WHEN** 60% of the last 72 h were at 15–25 °C with RH ≥ 90%
- **THEN** the gray-mold index is reported as high
