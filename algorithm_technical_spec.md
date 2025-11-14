# Algorithm Technical Specification
## Agricultural Equipment Rental Optimization

### Overview
This document provides the detailed technical specifications for implementing the three-tier algorithm system that delivers immediate, measurable value to agricultural equipment dealers.

---

## TIER 1: Lost Revenue Calculator

### Purpose
Quantify historical revenue loss due to equipment underutilization and poor positioning.

### Data Requirements

**Input Data:**
```json
{
  "equipment": {
    "id": "string",
    "type": "string",
    "category": "string",
    "acquisition_cost": "number",
    "location_id": "string"
  },
  "rental_history": [
    {
      "equipment_id": "string",
      "start_date": "date",
      "end_date": "date",
      "rental_rate": "number",
      "location_id": "string"
    }
  ],
  "maintenance_history": [
    {
      "equipment_id": "string",
      "start_date": "date",
      "end_date": "date",
      "maintenance_type": "string"
    }
  ],
  "missed_opportunities": [
    {
      "date": "date",
      "equipment_type": "string",
      "location_id": "string",
      "requested_by": "string"
    }
  ]
}
```

### Algorithm Implementation

**Step 1: Calculate Available Days**
```python
def calculate_available_days(equipment_id, start_date, end_date):
    """
    Calculate days equipment was available for rental
    """
    total_days = (end_date - start_date).days

    # Subtract days equipment was rented
    rented_days = sum([
        (rental.end_date - rental.start_date).days
        for rental in get_rentals(equipment_id, start_date, end_date)
    ])

    # Subtract days equipment was in maintenance
    maintenance_days = sum([
        (maint.end_date - maint.start_date).days
        for maint in get_maintenance(equipment_id, start_date, end_date)
    ])

    # Subtract days equipment was in transit
    transit_days = sum([
        (transfer.arrival_date - transfer.departure_date).days
        for transfer in get_transfers(equipment_id, start_date, end_date)
    ])

    available_days = total_days - rented_days - maintenance_days - transit_days

    return {
        'total_days': total_days,
        'rented_days': rented_days,
        'maintenance_days': maintenance_days,
        'transit_days': transit_days,
        'available_days': available_days,
        'utilization_rate': rented_days / total_days if total_days > 0 else 0
    }
```

**Step 2: Calculate Market Rate**
```python
def calculate_market_rate(equipment_type, location_id, date_range):
    """
    Determine market rental rate for equipment type
    """
    # Get historical rental rates for this equipment type
    historical_rates = get_historical_rates(equipment_type, date_range)

    # Get competitor rates (if available via scraping/API)
    competitor_rates = get_competitor_rates(equipment_type, location_id)

    # Calculate weighted average
    if competitor_rates:
        market_rate = (
            0.7 * median(historical_rates) +
            0.3 * median(competitor_rates)
        )
    else:
        market_rate = median(historical_rates)

    # Adjust for seasonality
    seasonal_modifier = get_seasonal_modifier(equipment_type, date_range)

    return market_rate * seasonal_modifier
```

**Step 3: Calculate Capture Probability**
```python
def calculate_capture_probability(equipment_id, available_days_list):
    """
    Estimate probability equipment would have been rented if positioned correctly
    """
    equipment = get_equipment(equipment_id)
    location = get_location(equipment.location_id)

    # Factor 1: Historical demand at this location for this equipment type
    local_demand_score = calculate_local_demand(
        equipment.type,
        equipment.location_id,
        available_days_list
    )

    # Factor 2: Demand at other locations (missed opportunities)
    network_demand_score = calculate_network_demand(
        equipment.type,
        available_days_list,
        exclude_location=equipment.location_id
    )

    # Factor 3: Seasonal demand pattern
    seasonal_demand = get_seasonal_demand_pattern(
        equipment.type,
        available_days_list
    )

    # Factor 4: Competitive availability
    competitive_factor = estimate_competitive_availability(
        equipment.type,
        location.region,
        available_days_list
    )

    # Weighted combination
    capture_probability = (
        0.40 * local_demand_score +      # Would rent at current location
        0.30 * network_demand_score +    # Could transfer to high-demand location
        0.20 * seasonal_demand +          # Time of year matters
        0.10 * competitive_factor         # Less competition = higher capture
    )

    # Cap at realistic maximum (never assume 100% capture)
    return min(capture_probability, 0.85)
```

**Step 4: Calculate Lost Revenue**
```python
def calculate_lost_revenue(equipment_id, start_date, end_date):
    """
    Main function to calculate lost revenue for a piece of equipment
    """
    # Get available days breakdown
    days_analysis = calculate_available_days(equipment_id, start_date, end_date)
    available_days = days_analysis['available_days']

    if available_days == 0:
        return {
            'lost_revenue': 0,
            'reason': 'Equipment fully utilized',
            'details': days_analysis
        }

    # Get market rate for this equipment
    equipment = get_equipment(equipment_id)
    market_rate = calculate_market_rate(
        equipment.type,
        equipment.location_id,
        (start_date, end_date)
    )

    # Get capture probability
    available_days_list = get_available_days_list(equipment_id, start_date, end_date)
    capture_probability = calculate_capture_probability(
        equipment_id,
        available_days_list
    )

    # Calculate lost revenue
    potential_revenue = available_days * market_rate
    lost_revenue = potential_revenue * capture_probability

    # Breakdown by reason
    reasons = analyze_lost_revenue_reasons(
        equipment_id,
        available_days_list,
        market_rate,
        capture_probability
    )

    return {
        'equipment_id': equipment_id,
        'equipment_type': equipment.type,
        'period': f"{start_date} to {end_date}",
        'available_days': available_days,
        'market_rate_per_day': market_rate,
        'capture_probability': capture_probability,
        'potential_revenue': potential_revenue,
        'lost_revenue': lost_revenue,
        'utilization_rate': days_analysis['utilization_rate'],
        'reasons': reasons,
        'recommendations': generate_recommendations(reasons)
    }
```

**Step 5: Analyze Reasons for Lost Revenue**
```python
def analyze_lost_revenue_reasons(equipment_id, available_days, market_rate, capture_prob):
    """
    Break down why revenue was lost
    """
    reasons = []

    # Reason 1: Wrong location
    demand_at_other_locations = get_demand_at_other_locations(
        equipment_id,
        available_days
    )
    if demand_at_other_locations:
        lost_to_location = sum([
            len(demand) * market_rate * capture_prob
            for demand in demand_at_other_locations.values()
        ])
        reasons.append({
            'reason': 'Wrong Location',
            'lost_revenue': lost_to_location,
            'details': demand_at_other_locations,
            'actionable': True,
            'action': 'Transfer to high-demand location'
        })

    # Reason 2: Poor marketing/visibility
    customer_inquiries = get_customer_inquiries_no_conversion(
        equipment_id,
        available_days
    )
    if customer_inquiries:
        lost_to_marketing = len(customer_inquiries) * market_rate * 0.5
        reasons.append({
            'reason': 'Low Visibility',
            'lost_revenue': lost_to_marketing,
            'details': customer_inquiries,
            'actionable': True,
            'action': 'Improve online listing, call previous customers'
        })

    # Reason 3: Pricing too high
    price_sensitivity = analyze_price_sensitivity(
        equipment_id,
        available_days,
        market_rate
    )
    if price_sensitivity['overpriced']:
        reasons.append({
            'reason': 'Price Too High',
            'lost_revenue': price_sensitivity['estimated_loss'],
            'details': price_sensitivity,
            'actionable': True,
            'action': f"Reduce price to ${price_sensitivity['recommended_price']}/day"
        })

    # Reason 4: Seasonal low demand (not actionable)
    seasonal_idle_days = identify_seasonal_idle_days(
        equipment_id,
        available_days
    )
    if seasonal_idle_days:
        lost_to_seasonality = len(seasonal_idle_days) * market_rate * 0.2
        reasons.append({
            'reason': 'Seasonal Low Demand',
            'lost_revenue': lost_to_seasonality,
            'details': seasonal_idle_days,
            'actionable': False,
            'action': 'Consider selling equipment or expanding service area'
        })

    # Reason 5: Equipment condition/age
    condition_issues = check_equipment_condition_impact(equipment_id)
    if condition_issues:
        reasons.append({
            'reason': 'Equipment Condition',
            'lost_revenue': condition_issues['estimated_loss'],
            'details': condition_issues,
            'actionable': True,
            'action': 'Repair or replace equipment'
        })

    return sorted(reasons, key=lambda x: x['lost_revenue'], reverse=True)
```

---

## TIER 2: Smart Transfer Optimizer

### Purpose
Calculate ROI of transferring equipment between locations and recommend optimal transfers.

### Data Requirements

**Additional Input Data:**
```json
{
  "locations": [
    {
      "id": "string",
      "name": "string",
      "address": "string",
      "lat": "number",
      "lon": "number"
    }
  ],
  "transport_costs": {
    "per_mile_rate": "number",
    "driver_hourly_rate": "number",
    "fuel_cost_per_mile": "number",
    "average_speed_mph": "number"
  },
  "pending_requests": [
    {
      "equipment_type": "string",
      "location_id": "string",
      "request_date": "date",
      "desired_start_date": "date",
      "desired_end_date": "date",
      "customer_id": "string"
    }
  ]
}
```

### Algorithm Implementation

**Step 1: Calculate Transport Cost**
```python
def calculate_transport_cost(from_location_id, to_location_id, equipment_id):
    """
    Calculate total cost of transferring equipment between locations
    """
    from_loc = get_location(from_location_id)
    to_loc = get_location(to_location_id)
    equipment = get_equipment(equipment_id)

    # Calculate distance
    distance_miles = calculate_distance(
        (from_loc.lat, from_loc.lon),
        (to_loc.lat, to_loc.lon)
    )

    # Get transport cost parameters
    costs = get_transport_costs()

    # Calculate components
    fuel_cost = distance_miles * costs['fuel_cost_per_mile']

    # Driver time (round trip)
    travel_time_hours = (distance_miles * 2) / costs['average_speed_mph']
    driver_cost = travel_time_hours * costs['driver_hourly_rate']

    # Equipment-specific transport cost
    # Heavier equipment costs more
    if equipment.requires_specialized_transport:
        specialized_transport_cost = distance_miles * costs['specialized_per_mile_rate']
        total_cost = specialized_transport_cost
    else:
        total_cost = fuel_cost + driver_cost + (distance_miles * costs['per_mile_rate'])

    # Add opportunity cost of driver time
    driver_opportunity_cost = travel_time_hours * costs['driver_alternative_value']

    return {
        'distance_miles': distance_miles,
        'fuel_cost': fuel_cost,
        'driver_cost': driver_cost,
        'transport_cost': total_cost,
        'travel_time_hours': travel_time_hours,
        'driver_opportunity_cost': driver_opportunity_cost,
        'total_cost_including_opportunity': total_cost + driver_opportunity_cost
    }
```

**Step 2: Estimate Revenue at Destination**
```python
def estimate_revenue_at_destination(equipment_id, destination_location_id, time_horizon_days=14):
    """
    Estimate revenue equipment would generate at destination location
    """
    equipment = get_equipment(equipment_id)

    # Check for pending requests at destination
    pending_requests = get_pending_requests(
        equipment.type,
        destination_location_id
    )

    # Guaranteed revenue from pending requests
    guaranteed_revenue = sum([
        calculate_rental_revenue(req)
        for req in pending_requests
    ])

    # Historical demand pattern at destination
    historical_pattern = get_historical_demand_pattern(
        equipment.type,
        destination_location_id,
        time_horizon_days
    )

    # Seasonal demand forecast
    forecast = forecast_demand(
        equipment.type,
        destination_location_id,
        datetime.now(),
        time_horizon_days
    )

    # Combine factors
    expected_rental_days = (
        0.5 * historical_pattern['avg_rental_days'] +
        0.3 * forecast['expected_rental_days'] +
        0.2 * len(pending_requests) * 3  # Assume avg 3-day rental
    )

    # Cap at time horizon
    expected_rental_days = min(expected_rental_days, time_horizon_days)

    # Calculate expected revenue
    market_rate = calculate_market_rate(
        equipment.type,
        destination_location_id,
        (datetime.now(), datetime.now() + timedelta(days=time_horizon_days))
    )

    expected_revenue = (guaranteed_revenue +
                       (expected_rental_days * market_rate * 0.75))  # 75% capture rate

    return {
        'guaranteed_revenue': guaranteed_revenue,
        'expected_rental_days': expected_rental_days,
        'market_rate': market_rate,
        'expected_revenue': expected_revenue,
        'confidence': calculate_confidence(pending_requests, historical_pattern, forecast)
    }
```

**Step 3: Calculate Opportunity Cost at Current Location**
```python
def calculate_opportunity_cost(equipment_id, current_location_id, time_horizon_days=14):
    """
    Estimate revenue that would be lost by moving equipment from current location
    """
    equipment = get_equipment(equipment_id)

    # Check pending requests at current location
    pending_local_requests = get_pending_requests(
        equipment.type,
        current_location_id
    )

    if pending_local_requests:
        # High opportunity cost - requests waiting
        return {
            'opportunity_cost': sum([calculate_rental_revenue(req) for req in pending_local_requests]),
            'reason': 'Pending requests at current location',
            'confidence': 'HIGH'
        }

    # Historical demand at current location
    historical_local = get_historical_demand_pattern(
        equipment.type,
        current_location_id,
        time_horizon_days
    )

    # Forecast demand at current location
    forecast_local = forecast_demand(
        equipment.type,
        current_location_id,
        datetime.now(),
        time_horizon_days
    )

    # Calculate expected lost revenue
    expected_local_rental_days = (
        0.6 * historical_local['avg_rental_days'] +
        0.4 * forecast_local['expected_rental_days']
    )

    market_rate_local = calculate_market_rate(
        equipment.type,
        current_location_id,
        (datetime.now(), datetime.now() + timedelta(days=time_horizon_days))
    )

    opportunity_cost = expected_local_rental_days * market_rate_local * 0.6  # 60% capture if stayed

    return {
        'opportunity_cost': opportunity_cost,
        'expected_rental_days': expected_local_rental_days,
        'reason': 'Potential local demand',
        'confidence': 'MEDIUM' if historical_local['data_points'] > 5 else 'LOW'
    }
```

**Step 4: Calculate Transfer ROI**
```python
def calculate_transfer_roi(equipment_id, from_location_id, to_location_id, time_horizon_days=14):
    """
    Calculate ROI of transferring equipment
    """
    # Get transport cost
    transport = calculate_transport_cost(from_location_id, to_location_id, equipment_id)

    # Get expected revenue at destination
    destination_revenue = estimate_revenue_at_destination(
        equipment_id,
        to_location_id,
        time_horizon_days
    )

    # Get opportunity cost at current location
    current_opportunity = calculate_opportunity_cost(
        equipment_id,
        from_location_id,
        time_horizon_days
    )

    # Calculate net ROI
    gross_benefit = destination_revenue['expected_revenue']
    total_cost = transport['total_cost_including_opportunity'] + current_opportunity['opportunity_cost']
    net_roi = gross_benefit - total_cost

    # Calculate ROI percentage
    roi_percentage = (net_roi / total_cost * 100) if total_cost > 0 else 0

    # Calculate payback period (days)
    if destination_revenue['market_rate'] > 0:
        payback_days = total_cost / destination_revenue['market_rate']
    else:
        payback_days = float('inf')

    # Determine priority
    priority = determine_transfer_priority(
        net_roi,
        destination_revenue,
        current_opportunity,
        payback_days
    )

    return {
        'equipment_id': equipment_id,
        'from_location': from_location_id,
        'to_location': to_location_id,
        'transport_cost': transport['transport_cost'],
        'transport_time_hours': transport['travel_time_hours'],
        'expected_revenue': destination_revenue['expected_revenue'],
        'opportunity_cost': current_opportunity['opportunity_cost'],
        'total_cost': total_cost,
        'net_roi': net_roi,
        'roi_percentage': roi_percentage,
        'payback_days': payback_days,
        'priority': priority,
        'confidence': destination_revenue['confidence'],
        'recommendation': 'TRANSFER' if net_roi > 500 else 'DO NOT TRANSFER',
        'breakdown': {
            'transport': transport,
            'destination': destination_revenue,
            'opportunity': current_opportunity
        }
    }
```

**Step 5: Prioritize Transfer Recommendations**
```python
def determine_transfer_priority(net_roi, destination_revenue, current_opportunity, payback_days):
    """
    Determine priority level for transfer recommendation
    """
    # URGENT: Customer waiting + high ROI
    if destination_revenue['guaranteed_revenue'] > 0 and net_roi > 1000:
        return {
            'level': 'URGENT',
            'reason': 'Customer waiting - high revenue opportunity',
            'color': 'red',
            'action_timeframe': 'Transfer today'
        }

    # HIGH: Strong ROI + short payback
    elif net_roi > 1500 and payback_days < 7:
        return {
            'level': 'HIGH',
            'reason': 'Excellent ROI with fast payback',
            'color': 'orange',
            'action_timeframe': 'Transfer within 2 days'
        }

    # MEDIUM: Positive ROI + reasonable payback
    elif net_roi > 500 and payback_days < 14:
        return {
            'level': 'MEDIUM',
            'reason': 'Good ROI opportunity',
            'color': 'yellow',
            'action_timeframe': 'Transfer within week'
        }

    # LOW: Marginal benefit
    elif net_roi > 0:
        return {
            'level': 'LOW',
            'reason': 'Small positive ROI',
            'color': 'green',
            'action_timeframe': 'Consider if convenient'
        }

    # DO NOT TRANSFER
    else:
        return {
            'level': 'DO NOT TRANSFER',
            'reason': f'Negative ROI (${net_roi:.2f})',
            'color': 'gray',
            'action_timeframe': 'Keep at current location'
        }
```

**Step 6: Network-Wide Transfer Optimization**
```python
def optimize_fleet_positioning(dealer_network):
    """
    Optimize equipment positioning across entire dealer network
    """
    recommendations = []

    # Get all equipment and locations
    all_equipment = get_all_equipment(dealer_network)
    all_locations = get_all_locations(dealer_network)

    # For each piece of equipment
    for equipment in all_equipment:
        current_location = equipment.location_id

        # Calculate transfer ROI to every other location
        transfer_options = []
        for location in all_locations:
            if location.id != current_location:
                roi = calculate_transfer_roi(
                    equipment.id,
                    current_location,
                    location.id
                )
                if roi['net_roi'] > 0:
                    transfer_options.append(roi)

        # Sort by ROI
        transfer_options.sort(key=lambda x: x['net_roi'], reverse=True)

        # Add best option to recommendations
        if transfer_options:
            best_transfer = transfer_options[0]
            if best_transfer['net_roi'] > 500:  # Minimum threshold
                recommendations.append(best_transfer)

    # Sort all recommendations by priority and ROI
    recommendations.sort(
        key=lambda x: (
            x['priority']['level'] == 'URGENT',
            x['priority']['level'] == 'HIGH',
            x['net_roi']
        ),
        reverse=True
    )

    # Check for conflicts (same equipment recommended to multiple places)
    # Keep only highest ROI recommendation per equipment
    seen_equipment = set()
    final_recommendations = []
    for rec in recommendations:
        if rec['equipment_id'] not in seen_equipment:
            final_recommendations.append(rec)
            seen_equipment.add(rec['equipment_id'])

    return final_recommendations
```

---

## TIER 3: Demand Forecasting Engine

### Purpose
Predict future equipment demand to enable proactive positioning and capacity planning.

### Data Requirements

**Additional Input Data:**
```json
{
  "weather_forecast": {
    "location_id": "string",
    "forecast_date": "date",
    "temperature_high": "number",
    "temperature_low": "number",
    "precipitation_chance": "number",
    "precipitation_amount": "number",
    "conditions": "string"
  },
  "crop_calendar": {
    "region": "string",
    "crop_type": "string",
    "planting_window_start": "date",
    "planting_window_end": "date",
    "harvest_window_start": "date",
    "harvest_window_end": "date"
  },
  "economic_indicators": {
    "date": "date",
    "commodity_prices": {},
    "fuel_prices": "number",
    "farm_income_index": "number"
  }
}
```

### Algorithm Implementation

**Step 1: Seasonal Pattern Analysis**
```python
def analyze_seasonal_patterns(equipment_type, location_id, years_history=3):
    """
    Extract seasonal demand patterns from historical data
    """
    # Get rental history for past N years
    rentals = get_rental_history(
        equipment_type,
        location_id,
        start_date=datetime.now() - timedelta(days=365*years_history)
    )

    # Group by week of year
    weekly_demand = defaultdict(list)
    for rental in rentals:
        week_of_year = rental.start_date.isocalendar()[1]
        weekly_demand[week_of_year].append(rental)

    # Calculate statistics for each week
    seasonal_pattern = {}
    for week in range(1, 53):
        week_rentals = weekly_demand.get(week, [])

        if week_rentals:
            rental_days = [
                (r.end_date - r.start_date).days
                for r in week_rentals
            ]
            revenue = [r.total_revenue for r in week_rentals]

            seasonal_pattern[week] = {
                'avg_rentals': len(week_rentals) / years_history,
                'avg_rental_days': statistics.mean(rental_days),
                'avg_revenue': statistics.mean(revenue),
                'max_demand': max([len(get_rentals_for_week(year, week))
                                  for year in range(years_history)]),
                'demand_volatility': statistics.stdev(rental_days) if len(rental_days) > 1 else 0
            }
        else:
            seasonal_pattern[week] = {
                'avg_rentals': 0,
                'avg_rental_days': 0,
                'avg_revenue': 0,
                'max_demand': 0,
                'demand_volatility': 0
            }

    return seasonal_pattern
```

**Step 2: Weather Impact Modeling**
```python
def calculate_weather_impact(equipment_type, weather_forecast):
    """
    Adjust demand forecast based on weather conditions
    """
    equipment_weather_sensitivity = {
        'excavator': {
            'precipitation_impact': -0.7,  # Heavy negative impact
            'temperature_impact': 0.1,
            'optimal_conditions': 'clear, dry'
        },
        'tractor': {
            'precipitation_impact': -0.3,  # Moderate negative
            'temperature_impact': -0.2,  # Extreme temps reduce use
            'optimal_conditions': 'dry, moderate temps'
        },
        'compact_track_loader': {
            'precipitation_impact': -0.4,
            'temperature_impact': -0.1,
            'optimal_conditions': 'dry, any temperature'
        }
    }

    sensitivity = equipment_weather_sensitivity.get(equipment_type, {
        'precipitation_impact': -0.5,
        'temperature_impact': -0.1,
        'optimal_conditions': 'favorable'
    })

    # Calculate weather modifier (0.0 to 2.0, where 1.0 is neutral)
    modifier = 1.0

    # Precipitation impact
    if weather_forecast['precipitation_chance'] > 50:
        precip_factor = weather_forecast['precipitation_chance'] / 100
        modifier += sensitivity['precipitation_impact'] * precip_factor

    # Temperature impact (extreme heat or cold)
    temp_avg = (weather_forecast['temperature_high'] +
                weather_forecast['temperature_low']) / 2

    if temp_avg < 32 or temp_avg > 95:
        temp_extreme = abs(temp_avg - 70) / 70  # Deviation from ideal
        modifier += sensitivity['temperature_impact'] * temp_extreme

    # Ensure modifier stays in reasonable range
    modifier = max(0.2, min(modifier, 1.5))

    return {
        'weather_modifier': modifier,
        'explanation': generate_weather_explanation(modifier, weather_forecast),
        'confidence': 'HIGH' if weather_forecast else 'LOW'
    }
```

**Step 3: Crop Calendar Integration**
```python
def get_agricultural_activity_impact(equipment_type, location_id, forecast_date):
    """
    Adjust demand based on agricultural calendar
    """
    # Get local crop calendar
    region = get_region(location_id)
    crop_calendar = get_crop_calendar(region)

    # Check if forecast date falls in critical ag periods
    activity_impact = 1.0
    activity_type = None

    for crop in crop_calendar:
        # Planting season
        if crop['planting_window_start'] <= forecast_date <= crop['planting_window_end']:
            if equipment_type in ['tractor', 'planter', 'field_cultivator']:
                activity_impact = max(activity_impact, 1.8)
                activity_type = f"{crop['crop_type']} planting"

        # Harvest season
        if crop['harvest_window_start'] <= forecast_date <= crop['harvest_window_end']:
            if equipment_type in ['combine', 'grain_cart', 'tractor']:
                activity_impact = max(activity_impact, 2.0)
                activity_type = f"{crop['crop_type']} harvest"

    return {
        'activity_modifier': activity_impact,
        'activity_type': activity_type,
        'confidence': 'HIGH'
    }
```

**Step 4: Leading Indicators**
```python
def calculate_leading_indicators_impact(equipment_type, location_id, forecast_date):
    """
    Use leading indicators to adjust demand forecast
    """
    indicators = {}
    impact_modifier = 1.0

    # Construction permits (for compact equipment)
    if equipment_type in ['skid_steer', 'compact_excavator', 'compact_track_loader']:
        permits = get_construction_permits(location_id, days_back=30)
        if permits:
            # More permits = higher demand for compact equipment
            permits_impact = min(len(permits) / 10, 1.5)
            impact_modifier *= (1.0 + permits_impact * 0.3)
            indicators['construction_permits'] = len(permits)

    # Commodity prices (affects farm income = affects equipment rental)
    commodity_prices = get_commodity_prices()
    price_trend = calculate_price_trend(commodity_prices, days_back=90)

    if price_trend > 0.1:  # Prices up 10%+
        impact_modifier *= 1.15  # Farmers have more income
        indicators['commodity_price_trend'] = 'rising'
    elif price_trend < -0.1:  # Prices down 10%+
        impact_modifier *= 0.90  # Farmers have less income
        indicators['commodity_price_trend'] = 'falling'

    # Fuel prices (affects operating costs)
    fuel_prices = get_fuel_prices()
    if fuel_prices['current'] > fuel_prices['avg_last_year'] * 1.2:
        # High fuel = more rentals (farmers avoid buying)
        impact_modifier *= 1.1
        indicators['fuel_price_impact'] = 'high - positive for rentals'

    return {
        'leading_indicators_modifier': impact_modifier,
        'indicators': indicators,
        'confidence': 'MEDIUM'
    }
```

**Step 5: Real-Time Signals**
```python
def incorporate_realtime_signals(equipment_type, location_id):
    """
    Use real-time signals to adjust short-term forecast
    """
    signals = {}
    modifier = 1.0

    # Current reservation pipeline
    pending_reservations = get_pending_reservations(equipment_type, location_id)
    if pending_reservations:
        # Strong signal of imminent demand
        modifier *= 1.5
        signals['pending_reservations'] = len(pending_reservations)

    # Recent customer inquiries (calls, website)
    recent_inquiries = get_customer_inquiries(
        equipment_type,
        location_id,
        days_back=7
    )
    if len(recent_inquiries) > 3:
        # Multiple inquiries = building demand
        modifier *= 1.2
        signals['recent_inquiries'] = len(recent_inquiries)

    # Competitor availability
    competitor_availability = check_competitor_availability(equipment_type, location_id)
    if competitor_availability['available_units'] < 2:
        # Limited competition = higher capture rate
        modifier *= 1.3
        signals['limited_competition'] = True

    # Recent cancellations (negative signal)
    recent_cancellations = get_recent_cancellations(equipment_type, location_id, days_back=7)
    if len(recent_cancellations) > 2:
        # Economic stress or demand softening
        modifier *= 0.85
        signals['recent_cancellations'] = len(recent_cancellations)

    return {
        'realtime_modifier': modifier,
        'signals': signals,
        'confidence': 'HIGH'  # Real-time data is most reliable
    }
```

**Step 6: Combined Demand Forecast**
```python
def forecast_demand(equipment_type, location_id, start_date, forecast_days=14):
    """
    Main demand forecasting function combining all factors
    """
    forecasts = []

    for day_offset in range(forecast_days):
        forecast_date = start_date + timedelta(days=day_offset)
        week_of_year = forecast_date.isocalendar()[1]

        # Get base seasonal pattern
        seasonal_patterns = analyze_seasonal_patterns(equipment_type, location_id)
        base_demand = seasonal_patterns[week_of_year]['avg_rentals'] / 7  # Daily average

        # Get modifiers
        weather_forecast = get_weather_forecast(location_id, forecast_date)
        weather_impact = calculate_weather_impact(equipment_type, weather_forecast)

        ag_impact = get_agricultural_activity_impact(equipment_type, location_id, forecast_date)

        leading_impact = calculate_leading_indicators_impact(equipment_type, location_id, forecast_date)

        # Real-time signals (more weight for near-term forecasts)
        realtime_impact = incorporate_realtime_signals(equipment_type, location_id)
        realtime_weight = max(0, 1.0 - (day_offset / 14))  # Decay over time

        # Combine all factors with weights
        forecast_multiplier = (
            0.40 * weather_impact['weather_modifier'] +
            0.30 * ag_impact['activity_modifier'] +
            0.15 * leading_impact['leading_indicators_modifier'] +
            0.15 * (realtime_impact['realtime_modifier'] * realtime_weight +
                    1.0 * (1 - realtime_weight))
        )

        # Calculate final forecast
        forecasted_demand = base_demand * forecast_multiplier

        # Calculate confidence based on data quality
        confidence_score = calculate_forecast_confidence(
            seasonal_patterns[week_of_year],
            weather_impact,
            ag_impact,
            leading_impact,
            realtime_impact,
            day_offset
        )

        forecasts.append({
            'date': forecast_date,
            'equipment_type': equipment_type,
            'location_id': location_id,
            'forecasted_rental_probability': min(forecasted_demand, 1.0),
            'base_seasonal_demand': base_demand,
            'weather_modifier': weather_impact['weather_modifier'],
            'agricultural_modifier': ag_impact['activity_modifier'],
            'leading_indicators_modifier': leading_impact['leading_indicators_modifier'],
            'realtime_modifier': realtime_impact['realtime_modifier'],
            'final_forecast': forecasted_demand,
            'confidence': confidence_score,
            'explanation': generate_forecast_explanation(
                base_demand, weather_impact, ag_impact, leading_impact, realtime_impact
            )
        })

    # Aggregate for summary
    total_forecasted_rental_days = sum([f['forecasted_rental_probability'] for f in forecasts])

    return {
        'equipment_type': equipment_type,
        'location_id': location_id,
        'forecast_period_days': forecast_days,
        'daily_forecasts': forecasts,
        'summary': {
            'total_expected_rental_days': total_forecasted_rental_days,
            'avg_confidence': statistics.mean([f['confidence'] for f in forecasts]),
            'peak_demand_dates': sorted(forecasts, key=lambda x: x['final_forecast'], reverse=True)[:3],
            'low_demand_dates': sorted(forecasts, key=lambda x: x['final_forecast'])[:3]
        }
    }
```

**Step 7: Forecast Accuracy Tracking & Learning**
```python
def track_forecast_accuracy_and_learn(equipment_type, location_id):
    """
    Track forecast accuracy and improve algorithm over time
    """
    # Get past forecasts
    past_forecasts = get_historical_forecasts(
        equipment_type,
        location_id,
        days_back=90
    )

    # Get actual rentals for same period
    actual_rentals = get_rental_history(
        equipment_type,
        location_id,
        days_back=90
    )

    accuracy_metrics = []

    for forecast in past_forecasts:
        # Find actual rental for this date
        actual = next(
            (r for r in actual_rentals if r.date == forecast['date']),
            None
        )

        forecasted_rental = forecast['forecasted_rental_probability']
        actual_rental = 1 if actual else 0

        # Calculate error
        error = abs(forecasted_rental - actual_rental)

        accuracy_metrics.append({
            'date': forecast['date'],
            'forecasted': forecasted_rental,
            'actual': actual_rental,
            'error': error,
            'error_squared': error ** 2
        })

    # Calculate overall accuracy
    mean_absolute_error = statistics.mean([m['error'] for m in accuracy_metrics])
    root_mean_squared_error = math.sqrt(statistics.mean([m['error_squared'] for m in accuracy_metrics]))

    # Identify which factors were most accurate
    factor_accuracy = analyze_factor_accuracy(accuracy_metrics, past_forecasts)

    # Adjust future forecast weights based on accuracy
    updated_weights = optimize_forecast_weights(factor_accuracy)

    return {
        'accuracy_metrics': {
            'mean_absolute_error': mean_absolute_error,
            'rmse': root_mean_squared_error,
            'accuracy_percentage': (1 - mean_absolute_error) * 100
        },
        'factor_performance': factor_accuracy,
        'recommended_weight_adjustments': updated_weights,
        'forecast_quality': 'GOOD' if mean_absolute_error < 0.2 else 'NEEDS_IMPROVEMENT'
    }
```

---

## Integration & Output Specifications

### Dashboard Output Format

**Lost Revenue Report:**
```json
{
  "report_period": "2024-Q1",
  "total_lost_revenue": 127450,
  "total_fleet_value": 2500000,
  "current_utilization": 0.42,
  "target_utilization": 0.55,
  "equipment_breakdown": [
    {
      "equipment_id": "EQ-045",
      "equipment_type": "John Deere 310L Backhoe",
      "lost_revenue": 21150,
      "days_idle": 47,
      "reasons": [
        {
          "reason": "Wrong Location",
          "impact": 15000,
          "actionable": true
        }
      ],
      "recommendation": "Transfer to Location B"
    }
  ],
  "quick_wins": [
    "Transfer 3 units to high-demand locations: $42K potential revenue",
    "Adjust pricing on 5 underperforming units: $18K potential revenue",
    "Improve marketing for 4 invisible units: $12K potential revenue"
  ]
}
```

**Transfer Recommendations:**
```json
{
  "recommendations": [
    {
      "priority": "URGENT",
      "equipment_id": "EQ-023",
      "equipment_name": "Skid Steer #23",
      "from_location": "Store A",
      "to_location": "Store B",
      "net_roi": 2250,
      "transport_cost": 450,
      "expected_revenue": 3500,
      "payback_days": 3,
      "reason": "Customer waiting - 2 pending requests",
      "confidence": "HIGH"
    }
  ]
}
```

**Demand Forecast:**
```json
{
  "forecast_period": "Next 14 days",
  "equipment_summary": [
    {
      "equipment_type": "Compact Track Loader",
      "owned_units": 3,
      "forecasted_demand_days": 18,
      "available_capacity_days": 42,
      "status": "OPTIMAL",
      "action": "None needed"
    },
    {
      "equipment_type": "Backhoe",
      "owned_units": 2,
      "forecasted_demand_days": 22,
      "available_capacity_days": 14,
      "status": "SHORTAGE",
      "lost_revenue_estimate": 3600,
      "action": "Transfer 1 unit from another location OR rent from competitor"
    }
  ]
}
```

---

## Performance Requirements

**Response Time:**
- Lost Revenue Calculator: < 2 seconds for 100 equipment items
- Transfer Optimizer: < 5 seconds for network-wide analysis
- Demand Forecast: < 3 seconds for 14-day forecast

**Accuracy Targets:**
- Demand Forecast: 80%+ accuracy within 7 days
- Transfer ROI: 85%+ of recommendations should be profitable
- Lost Revenue: ±15% margin of error acceptable

**Scalability:**
- Support up to 500 equipment items per dealer
- Support up to 50 locations per dealer
- Process 3 years of historical data

---

## Data Privacy & Security

**Requirements:**
- All dealer data encrypted at rest and in transit
- No cross-dealer data sharing without explicit permission
- Aggregate anonymous data for industry benchmarks
- GDPR/CCPA compliant data handling

**Competitive Intelligence:**
- Competitor rate scraping done ethically (public websites only)
- No unauthorized access to competitor systems
- Respect robots.txt and rate limiting

---

This technical specification provides the complete algorithm methodology for immediate implementation. Each tier builds on the previous, creating a system that dealers can trust, verify, and profit from immediately.
