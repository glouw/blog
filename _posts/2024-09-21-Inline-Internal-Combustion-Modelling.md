---
layout: post
---

In this post I outline one way to simulate a four stroke Inline Internal Combustion Engine (IICE) with
realistic and performant procedural audio. The engine core utilizes fundamentals of thermodynamics,
fluid mechanics, classical dynamics, and (should you deviate from the above), high school chemistry.

Applications include, but are not limited to, game development vehicular audio and physics:

// -> EMBED VIDEO HERE OF INLINE 5

## The Piston and Classical Dynamics

An inline piston's movement is constrained by its vertical axis. Updating a piston's theta (in radians)
updates the piston pin's x and y position (in meters) and its bearing x and y position (in meters).
The conrod (connecting rod) length (in meters) and crank throw (in meters) are effectively constants.

```
struct piston
{
    double pin_x_m;
    double pin_y_m;
    double bearing_x_m;
    double bearing_y_m;
    double theta_r;
    double conrod_m;
    double conrod_crank_throw_m;
}
```

The piston updates with an applied angular velocity (in radians per second). The delta radians
applied to the piston's theta is determined by the time step which is effectively the inverse of
a common audio sampling rate of 44100 Hz.

```
#define DT_S (1.0 / 44100.0)

void
update_position(struct piston* self, const double angular_velocity_r_per_s)
{
    self->theta_r += angular_velocity_r_per_s * DT_S;
    self->bearing_x_m = self->conrod_crank_throw_m * sin(self->theta_r);
    self->bearing_y_m = self->conrod_crank_throw_m * cos(self->theta_r);
    self->pin_x_m = 0.0;
    self->pin_y_m = sqrt(pow(self->conrod_m, 2.0) - pow(self->conrod_crank_throw_m * sin(self->theta_r), 2.0)) + self->conrod_crank_throw_m * cos(self->theta_r);
}
```

## The Piston and the Otto Cycle

The piston provides engine torque by igniting a compressed air fuel mixture. This process
is known as the Otto cycle:

A: Piston inlet opens.
B: Piston chamber expands. Cool atmospheric air draws in.
C: Piston inlet closes.
D: Piston chamber compresses. Air temperature and pressure raises due to compression.
E: Fuel is injected. Air to Fuel ratio is then roughly 14.7 parts air to 1 part fuel.
F: Air Fuel Mixture is combusted, creating a downwards force on the piston head.
G: Downwards force creates torque, causing the piston chamber to expand, cooling the piston gas and lowering pressure.
H: Piston outlet opens. Hot piston air expels to the atmosphere.
I: Piston outlet closes.
J: The cycle repeats.

// -> SHOW DEMO OF IICE SINGLE PISTON

The input of the Otto cycle relies on its output. That is to say, chamber expansion and compression in
step B and D require a mechanical force provided by the torque created in step G.
Since no system is truly perpetual, a secondary _starter_ motor with a large gear ratio
is tacked on to provide the initial compression and decompression torque. The starter motor then disconnects.

Piston inlet and outlet openness (lift) ratio is a function of piston theta, where `engage_r` (in radians)
determines at which point valve opening begins, and `ramp_r` the ramp length:

```
double
calc_lift_ratio(double theta_r, double engage_r, double ramp_r)
{
    double theta_mod_four_stroke_r = calc_theta_mod_four_stroke_r(theta_r);
    if(theta_mod_four_stroke_r > engage_r)
    {
        double open_r = theta_mod_four_stroke_r - engage_r;
        double term1 = 35.0 * pow(open_r / ramp_r, 4.0);
        double term2 = 84.0 * pow(open_r / ramp_r, 5.0);
        double term3 = 70.0 * pow(open_r / ramp_r, 6.0);
        double term4 = 20.0 * pow(open_r / ramp_r, 7.0);
        return term1 - term2 + term3 - term4;
    }
    return 0.0;
}
```

Where four stroke modulation is calculated as:

```
#define double FOUR_STROKE_R (4.0 * M_PI)

double
calc_theta_mod_four_stroke_r(double theta_r)
{
    return fmod(theta_r, FOUR_STROKE_R);
}
```

The Otto cycle can similarly be described with a pressure volume diagram:

// -> DRAW PV DIAGRAM HERE

While our model _can_ be a piston connected to an atmospheric source and sink, engine sound and performance is greatly
improved upon by adding intake and exhaust chambers. The intake chamber is a lump sum of the throttle chamber, plenum,
and intake runner, for simplicity's sake. Similarly, the exhaust chamber is a lump sum of the exhaust runner, collector,
tailpipe, muffler. The lump sum can be further broken down into individual chamber components to achieve more accurate
engine sounds and performance.

// -> CHAMBERS GO HERE

## Chamber Definition

Chambers are comprised of a volume (in cubic meters) and a gas:

```
struct chamber
{
    double volume_m3;
    struct gas gas;
};

```

A gas at rest is comprised of some mole count (in moles) and static temperature (in kelvins).
The mole count specifies the amount of substance, (think gas molecules), physically present.
The static temperature represents the average kinetic energy of the substance (think gas molecules),
moving in any random direction, while also being at rest in bulk to the observer.

```
struct gas
{
    double static_temperature_k;
    double moles;
};
```

The static pressure of the gas is derived from the ideal gas law:

```
#define R_J_PER_MOL_K 8.314

double
calc_static_pressure_pa(struct chamber* self)
{
    return (self->gas.moles * R_J_PER_MOL_K * self->gas.static_temperature_k) / self->volume_m3;
}
```

A chamber's static pressure represents the average force of molecules against the inner
walls of a chamber. Like with the static temperature definition, the velocity (in meters per second)
of each gas molecules is random, and so the net _bulk velocity_ of the gas is zero; the gas does not
move in any particular direction like a cool breeze on a warm summer's day. IICE's are
concerned with the movement of gas from intake to exhaust, and so a secondary one dimensional description
of the moving gas in terms of bulk momentum (in kilogram meters per second) is needed to describe the corollary
to static pressure and static temperature - total pressure and total temperature.

```
struct gas
{
    double static_temperature_k;
    double moles;
+   double bulk_momentum_kg_m_per_s;
};
```

Although to calculate total pressure and total temperature, we are required to compute the mass
of the gas and the bulk velocity of the gas from the bulk momentum of the gas. The
mass of the gas requires that we know the molar mass (in kilogram per mol) of the gas. From the
Otto cycle, the gasses present within any chamber are lumped into either air, fuel, or combusted fuel.
Their specific molar masses are defined as:

```
#define MOLAR_MASS_COMBUSTED_KG_PER_MOL 0.02885
#define MOLAR_MASS_AIR_KG_PER_MOL 0.0289647
#define MOLAR_MASS_FUEL_KG_PER_MOL 0.11423
```

Updating the gas definition:

```
struct gas
{
    double static_temperature_k;
    double moles;
    double bulk_momentum_kg_m_per_s;
+   double combusted_ratio;
+   double air_ratio;
+   double fuel_ratio;
};
```

The molar mass of the chamber is then calculated as the sum of three:

```
double
calc_molar_mass_kg_per_mol(struct chamber* self)
{
    return self->gas.combusted_ratio * MOLAR_MASS_COMBUSTED_KG_PER_MOL
         + self->gas.air_ratio * MOLAR_MASS_AIR_KG_PER_MOL
         + self->gas.fuel_ratio * MOLAR_MASS_FUEL_KG_PER_MOL;
}
```

From the molar mass we can derive gas mass (in kilograms):

```
double
calc_mass_kg(struct chamber* self)
{
    return self->gas.moles * calc_molar_mass_kg_per_mol(self);
}
```

And gas density (in kilogram per meter cubed):

```
double
calc_density_kg_per_m3(struct chamber* self)
{
    return calc_mass_kg(self) / self->volume_m3;
}
```

And gas bulk velocity (in meters per second):

```
double
calc_bulk_velocity_m_per_s(struct chamber* self)
{
    return self->gas.bulk_momentum_kg_m_per_s / calc_mass_kg(self);
}
```

The dynamic pressure of the gas (in kelvin) is then defined as:

```
double
calc_dynamic_pressure_pa(struct chamber* self)
{
    return calc_density_kg_per_m3(self) * pow(calc_bulk_velocity_m_per_s(self), 2.0) / 2.0;
}
```

The total pressure of the gas the sum of dynamic and static pressure:

```
double
calc_total_pressure_pa(struct chamber* self)
{
    return calc_static_pressure_pa(self) + calc_dynamic_pressure_pa(self);
}
```

And the total temperature of the gas, while knowing the gamma of the gas, is derived as:

```
double
calc_total_temperature_k(struct chamber* self, double mach_number)
{
    return self->gas.static_temperature_k * (1.0 + (calc_gamma(self) - 1.0) / 2.0 * pow(mach_number, 2.0));
}
```

The gas gamma value (no dimensions) characterizes how a gas behaves during compression.
A higher gamma gas generally compresses more easily than a lower gamma gas,
and raises temperature greater than a lower gamma gas, under the same compression scenario.
The gamma calculation is similar to that of that molar mass calculation:

```
#define GAMMA_COMBUSTED 1.3
#define GAMMA_FUEL 1.15
#define GAMMA_AIR 1.4

double
calc_gamma(struct chamber* self)
{
    return self->gas.combusted_ratio * GAMMA_COMBUSTED
         + self->gas.air_ratio * GAMMA_AIR
         + self->gas.fuel_ratio * GAMMA_FUEL;
}
```

Compressing, or decompressing, the chamber to some new volume will respectively increase, or decrease,
the static temperature of the gas:

```
void
compress_adiabatically(struct chamber* self, double new_volume_m3)
{
    self->gas.static_temperature_k *= pow(self->volume_m3 / new_volume_m3, calc_gamma(&self->gas) - 1.0);
    self->volume_m3 = new_volume_m3;
}
```

## Mach Number and Mass Flow Rates

Flow from chamber to chamber is dictated by the mach number, as introduced as `mach_number` in the
total temperature function. A mach number less than 1.0 indicates that the flow is subsonic,
and a mach number equal to 1.0 indicates that the flow is choked, or sonic. For purposes of IICE
design, mach numbers greater than 1.0 indicated supersonic flow and will not be used. Clamping between 0.0 and 1.0 can be
applied to the mach number calculation:

```
double
calc_mach_number(double total_pressure_upstream_pa, double total_pressure_downstream_pa, double gamma_downstream)
{
    double compression_ratio = total_pressure_upstream_pa / total_pressure_downstream_pa;
    return sqrt((pow(compression_ratio, gamma_downstream / (gamma_downstream - 1.0)) - 1.0) * (2.0 / (gamma_downstream - 1.0)))
}
```

Flow always occurs from the higher total pressure chamber to the lower total pressure chamber. An increase in cross sectional area
between two chambers will linearly increase mass flow rate under steady state conditions:

```
double
calc_mass_flow_rate_kg_per_s(struct chamber* upstream, double mach_number, double cross_sectional_flow_area_m2)
{
    double term1 = cross_sectional_flow_area_m2 * calc_total_pressure_pa(upstream) / sqrt(calc_total_temperature_k(upstream, mach_number));
    double term2 = sqrt(calc_gamma(upstream) / calc_specific_gas_constant_j_per_kg_k(upstream)) * mach_number;
    double term3 = pow(1.0 + (calc_gamma(upstream) - 1.0) / 2.0 * pow(mach_number, 2.0), - (calc_gamma(upstream) + 1.0) / (2.0 * (calc_gamma(upstream) - 1.0)));
    return term1 * term2 * term3;
}
```

Where the gas constant (in joules per kilogram kelvin) is defined as:

```
double
calc_specific_gas_constant_j_per_kg_k(struct chamber* self)
{
    return R_J_PER_MOL_K / calc_molar_mass_kg_per_mol(self);
}
```

## Chamber Mass Transfer

The instantaneous velocity of the gas in the nozzle is derived from the mass flow rate, and total temperature and total pressure of the flow:
```
double
calc_velocity_m_per_s(struct chamber* self, double mass_flow_rate_kg_per_s, double mach_number, double cross_sectional_flow_area_m2)
{
    double term1 = mass_flow_rate_kg_per_s * calc_specific_gas_constant_j_per_kg_k(self) * calc_total_temperature_k(self, mach_number);
    double term2 = calc_total_pressure_pa(self) * cross_sectional_flow_area_m2;
    return term1 / term2;
}
```

The instantaneous velocity does not dictate the bulk velocity of the gas in the upstream or downstream chamber, but serves to
conserve momentum between the upstream and downstream chamber.

The mass flowed from chamber to chamber is then:
```
double mass_flowed_k = mass_flow_rate_kg_per_s * DT_S;
```

And the bulk momentum flowed from chamber to chamber is then:
```
double bulk_momentum_flowed_kg_m_per_s = mass_flowed_kg * velocity_m_per_s;
```

In a flow function, where flow always occurs from x to y, where x is the chamber with higher total pressure:
```
void
flow(struct chamber* x, struct chamber* y, double cross_sectional_flow_area_m2);
```

The bulk moles transferred from x to y is defined as:
```
double moles_flowed = mass_flowed_kg / calc_molar_mass_kg_per_mol(x);
```

And the moles of the two chambers update like:

```
add_moles(x, -moles_flowed);
add_moles(y, +moles_flowed);
```

Where mole addition is defined as:
```
void
add_moles(struct chamber* self, const double delta_moles)
{
    double new_moles = self->gas.moles + delta_moles;
    double new_static_temperature_k = self->gas.static_temperature_k * pow(new_moles / self->gas.moles, (calc_gamma(self) - 1.0) / calc_gamma(self));
    self->gas.static_temperature_k = new_static_temperature_k;
    self->gas.moles = new_moles;
}
```
Note that adding moles to a chamber increases the chamber's static temperature, which in turn increases static pressure.

Momentum between chambers is then conserved by:
```
x->gas.bulk_momentum_kg_m_per_s -= bulk_momentum_flowed_kg_m_per_s
y->gas.bulk_momentum_kg_m_per_s += bulk_momentum_flowed_kg_m_per_s
```

And the downstream chamber, assuming flow always occurs from x to y, has its air, fuel, and combusted ratios mixed with the inflowing gas:

```
y->gas.air_ratio = calc_weighted_average(y->gas.air_ratio, y->gas.moles, x->gas.air_ratio, moles_flowed);
y->gas.fuel_ratio = calc_weighted_average(y->gas.fuel_ratio, y->gas.moles, x->gas.fuel_ratio, moles_flowed);
y->gas.combusted_ratio = calc_weighted_average(y->gas.combusted_ratio, y->gas.moles, x->gas.combusted_ratio, moles_flowed);
```

The upstream chamber is not mixed.

Thermal cooling (or heating) of the downstream gas occurs as well:

```
y->gas.static_temperature_k = calc_weighted_average(y->gas.static_temperature_k, y->gas.moles, x->gas.static_temperature_k, moles_flowed);
```

Where the weighted average function is defined as:

```
double
calc_weighted_average(const double value1, const double weight1, const double value2, const double weight2)
{
    return (value1 * weight1 + value2 * weight2) / (weight1 + weight2);
}
```

## Direct Fuel Injection and Combustion

Direct fuel injection can be simplified to a lump addition model of gas moles. The fuel
is added instaneously, at a balanced 14.7 parts air to 1 parts fuel, and the air, fuel,
and combusted ratios are updated:

```
void
inject_directly(struct chamber* self)
{
    double air_fuel_ratio = 14.7;
    double current_air_moles = self->gas.moles * self->gas.air_ratio;
    double current_fuel_moles = self->gas.moles * self->gas.fuel_ratio;
    double current_combusted_moles = self->gas.moles * self->gas.combusted_ratio;
    double delta_fuel_moles = current_air_moles / air_fuel_ratio - current_fuel_moles;
    double new_air_moles = current_air_moles;
    double new_fuel_moles = current_fuel_moles + delta_fuel_moles;
    double new_combusted_moles = current_combusted_moles;
    add_moles_adiabatically(self, delta_fuel_moles);
    air_ratio = new_air_moles / self->gas.moles;
    fuel_ratio = new_fuel_moles / self->gas.moles;
    combusted_ratio = new_combusted_moles / self->gas.moles;
}
```

With a chamber primed with injected fuel, and the chamber ignited from the top,
the rate at which the combustion flame burns in the x and y direction (in meters per second)
is defined as:

```
#define STP_PRESSURE_PA 101325.0
#define STP_TEMPERATURE_K 273.15

double
calc_flame_speed_m_per_s(struct chamber* self)
{
    double pressure_exponent = 1.7;
    double temperature_exponent = 1.2;
    double laminar_flame_speed_m_per_s = 0.4;
    return laminar_flame_speed_m_per_s
        * pow(calc_static_pressure_pa(self) / STP_PRESSURE_PA, pressure_exponent)
        * pow(self->static_temperature_k / STP_TEMPERATURE_K, temperature_exponent);
}
```

Knowing the volume of the gas chamber burned for time step `DT_S`, the burned ratio is defined as:

```
double burned_ratio
{
    volume_burned_m3 / self->volume_m3
};
```

And the delta change in static temperature is defined as:

```
#define GASOLINE_LOWER_HEATING_VALUE_J_PER_KG 44e6

double
calc_delta_static_temperature_from_burned_afr_k(struct chamber* self, double burned_ratio)
{
    double burned_fuel_moles = self->fuel_ratio * self->moles * burned_ratio;
    double burned_fuel_mass_kg = burned_fuel_moles * calc_molar_mass_kg_per_mol(self);
    double energy_released_j = burned_fuel_mass_kg * GASOLINE_LOWER_HEATING_VALUE_J_PER_KG;
    return energy_released_j / (calc_mass_kg(self) * calc_specific_heat_capacity_at_constant_pressure_j_per_kg_k(self));
}
```

Where:

```
double
calc_specific_heat_capacity_at_constant_volume_j_per_kg_k(struct chamber* self)
{
    return calc_specific_gas_constant_j_per_kg_k(self) / (calc_gamma(self) - 1.0);
}

double
calc_specific_heat_capacity_at_constant_pressure_j_per_kg_k(self)
{
    return calc_gamma(self) * calc_specific_heat_capacity_at_constant_volume_j_per_kg_k(self);
}
```

The maximum allowable flame temperature is determined by the adiabatic flame tepmerature. Delta static temperature
changes to chamber static pressure may not exceed the adiabatic flame temperature.
For model simplification, the standard atmospheric temperature is used as an approximation for intitial temperature:

```
double
calc_adiabatic_flame_static_temperature_k(struct chamber* self)
{
    double air_fuel_ratio = self->air_ratio / self->fuel_ratio;
    double energy_density_j_per_kg = GASOLINE_LOWER_HEATING_VALUE_J_PER_KG / (1.0 + air_fuel_ratio);
    return STP_TEMPERATURE_K + energy_density_j_per_kg / calc_specific_heat_capacity_at_constant_pressure_j_per_kg_k(self);
}
```

## Piston Torque Generation

The added static temperature change from combustion directly raises the static pressure of the piston chamber.

Modifying our piston struct to support a combustion chamber:

```
struct piston
{
    double pin_x_m;
    double pin_y_m;
    double bearing_x_m;
    double bearing_y_m;
    double theta_r;
    double conrod_m;
    double conrod_crank_throw_m;
+   struct chamber chamber;
}
```

The chamber volume and gas states are tracked internally and updated per frame. The torque produced
by the piston (in newton meters) is defined as:

```
double
calc_gas_torque_nm(struct piston* self)
{
    double area_m2 = calc_circle_area_m2(self->head_radius_m);
    double term1 = calc_static_gauge_pressure_pa(&self->chamber) * area_m2 * self->conrod_crank_throw_m * sin(self->theta_r);
    double term2 = 1.0 + (self->conrod_crank_throw_m / self->conrod_m) * cos(self->theta_r);
    return term1 * term2;
}
```

Where:

```
double
calc_circle_area_m2(double radius_m)
{
    return M_PI * pow(radius_m, 2.0);
}
```

```
double
calc_static_gauge_pressure_pa(struct chamber* self)
{
    return calc_static_pressure_pa(self) - STP_PRESSURE_PA;
}
```

A piston moving at a high speed, especially a piston head with decent mass, creates inertia torque.
Modifying our piston:

```
struct piston
{
    double pin_x_m;
    double pin_y_m;
    double bearing_x_m;
    double bearing_y_m;
    double theta_r;
    double conrod_m;
    double conrod_crank_throw_m;
    struct chamber chamber;
+   double conrod_mass_kg;
+   double head_mass_kg;
}
```

The moment of inertia (in kilograms per meters squared) can be simplified as that of a
reciprocating mass:

```
double
calc_moment_of_inertia_kg_per_m2(struct piston* self)
{
    double mass_reciprocating_kg = self->head_mass_kg + 0.5 * self->conrod_mass_kg;
    double term1 = 1.0;
    double term2 = self->conrod_crank_throw_m / (4.0 * self->conrod_m);
    double term3 = pow(self->conrod_crank_throw_m, 2.0) / (8.0 * pow(self->conrod_m, 2.0));
    return mass_reciprocating_kg * pow(self->conrod_crank_throw_m, 2.0) * (term1 + term2 + term3);
}
```

And the inertia torque:

```
double
calc_inertia_torque_nm(struct piston* self, double angular_velocity_r_per_s)
{
    double term1 = 0.25 * sin(1.0 * self->theta_r) * self->conrod_crank_throw_m / self->conrod_m;
    double term2 = 0.50 * sin(2.0 * self->theta_r);
    double term3 = 0.75 * sin(3.0 * self->theta_r) * self->conrod_crank_throw_m / self->conrod_m;
    return calc_moment_of_inertia_kg_per_m2(self) * pow(angular_velocity_r_per_s, 2.0) * (term1 - term2 - term3);
}
```

The total torque produced by the piston is then:

```
double total_torque_nm = calc_moment_of_inertia_kg_per_m2(piston) + calc_inertia_torque_nm(piston, angular_velocity_r_per_s);
```

And the angular acceleration (in radians per second squared) supplied to the engine is:

```
double angular_acceleration_r_per_s2 = total_torque_nm / total_engine_moment_of_inertia_kg_per_m2;
```

Where the total engine moment of inertia accounts for the flywheel mass and radius and the moment of inertia of the piston.

```
double flywheel_moment_of_inertia_kg_per_m2 = 0.5 * mass_flywheel_kg * pow(radius_flywheel_m, 2.0);
```

Angular velocity is then updated by the angular acceleration time step:

```
angular_velocity_r_per_s += angular_acceleration_r_per_s2 * DT_S;
```

For IICE with more than one piston, the torques and moment of inertias for each piston are simply added together.
