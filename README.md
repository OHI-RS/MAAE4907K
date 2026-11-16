# MAAE4907K
Arduino Uno
// =====================================================================
// CSP Temperature Monitor
// Reads 3x DS18B20 sensors (tank, HEX inlet, HEX outlet), shows them on
// two 16x2 LCDs, logs CSV over serial, and drives a bypass valve relay.
// Code logic is unchanged from the original; only comments were added.
// =====================================================================

// ---- LIBRARIES ----
#include <LiquidCrystal.h>      // Drives the 16x2 character LCDs (HD44780 controller) in 4-bit mode
#include <OneWire.h>            // Low-level 1-Wire protocol (timing, reset, read/write bits)
#include <DallasTemperature.h>  // High-level DS18B20 functions built on OneWire (convert, read in C)


// -------- ONE WIRE BUS CONFIGURATION --------
// Connect all DS18B20 data pins to Pin 2.
// Requires a 5.1k ohm pull-up resistor from the data line to the 5V rail
// (the bus is open-drain, so without the pull-up the line floats).
// -------- ONE WIRE BUS --------
#define ONE_WIRE_BUS 2                  // Replace every ONE_WIRE_BUS with 2: the Arduino pin the sensor data line uses
OneWire oneWire(ONE_WIRE_BUS);          // Create a OneWire object that communicates on pin 2
DallasTemperature sensors(&oneWire);    // Create the temperature interface; it uses the OneWire object above (passed by address)

// -------- LCD CONFIG --------
// LiquidCrystal(RS, E, D4, D5, D6, D7)

// -------- LCD PIN MAPPING (SHARED BUS) --------
// Both LCDs share the same RS and Data lines to save Arduino pins.
// LiquidCrystal(RS, E, D4, D5, D6, D7)
// Shared: RS=12, D4=A0, D5=A1, D6=A2, D7=A3
// Unique: Enable (E) pins separate the two screens.
// Only the screen whose Enable pin gets pulsed latches the data, so the
// shared lines can address each LCD individually.

LiquidCrystal lcdTank(12, 11, A0, A1, A2, A3); // Tank Display: RS=12, E=11, data on A0-A3 (analog pins used as digital)
LiquidCrystal lcdHEX(12, 10, A0, A1, A2, A3);  // HEX Display: same RS and data pins, but E=10

//-------Valve.Control.Setup---------
int ValvePower = 8;   // Arduino pin 8 drives the solid-state relay that powers the motorized 3-way valve

void setup() {        // setup() runs once when the Arduino powers up or resets
  Serial.begin(9600); // Start serial communication at 9600 baud for data logging (CSV format)

  // ---- CSV HEADER ----
  // This makes the columns easy to identify when you import into Excel
  Serial.println("Tank_Temp(C),HEX_In(C),HEX_Out(C)"); // Send the column names as the first line, then a newline

  sensors.begin();    // Initialize One-Wire sensors (scans the bus, counts and records the sensors found)
  lcdTank.begin(16, 2); // Initialize the tank LCD as 16 columns x 2 rows
  lcdHEX.begin(16, 2);  // Initialize the HEX LCD as 16 columns x 2 rows

  // Static labels (printed once to save processing time in the loop)
  lcdTank.print("TANK TEMPERATURE"); // Print the tank title on row 0 (the cursor starts at column 0, row 0). Exactly 16 characters
  lcdHEX.setCursor(0,0);             // Move the HEX LCD cursor to column 0, row 0
  lcdHEX.print("HEX_IN: ");          // Label for the inlet temperature (8 characters, so values start at column 8)
  lcdHEX.setCursor(0,1);             // Move the cursor to column 0, row 1
  lcdHEX.print("HEX_OUT: ");         // Label for the outlet temperature (9 characters, so values start at column 9)

  //Set.Pin.Output.0.to.begin
  // NOTE: pinMode(ValvePower, OUTPUT) is not called anywhere in this sketch.
  // Without it, pin 8 stays an input and digitalWrite() only toggles its
  // internal pull-up, so the relay may not switch properly.
  digitalWrite(ValvePower, LOW);     // Start with pin 8 LOW (relay off, valve in its default position)

}

void loop() {         // loop() runs repeatedly, forever, after setup() finishes
  // Command all sensors on the bus to take a reading
  sensors.requestTemperatures();     // Broadcast a "convert temperature" command to every DS18B20 and wait for the conversion to finish

  float T_tank = sensors.getTempCByIndex(0); // Temperature in C of the 1st sensor found on the bus (assumed to be the tank)
  float T_in   = sensors.getTempCByIndex(1); // 2nd sensor on the bus (assumed to be the HEX inlet)
  float T_out  = sensors.getTempCByIndex(2); // 3rd sensor on the bus (assumed to be the HEX outlet)
  // Index order depends on each sensor's unique address, not its physical location.
  // A sensor that is missing or not responding returns -127.0.

  //ValveControl
  if (T_out < (T_tank - 2)){         // Outlet is more than 2 C COLDER than the tank: heated water is not hot enough to return
    digitalWrite(ValvePower, HIGH);  // Pin 8 HIGH: relay closes and the valve is switched
    //start.bypass                   // Water recirculates through the bypass loop until it heats up
  }
  else if (T_out > (T_tank + 2)){    // Outlet is more than 2 C HOTTER than the tank: water is worth returning to the tank
    digitalWrite(ValvePower,LOW);    // Pin 8 LOW: relay opens and the valve switches back
    //no.bypass.loop                 // Flow goes to the tank
  }
  else{}                             // Within +/-2 C of the tank (deadband): do nothing and keep the last valve state so the motor does not rapidly toggle

  // ---- Tank LCD ----
  lcdTank.setCursor(0,1);    // Move to second row, column 0 (the title stays on row 0)
  lcdTank.print(T_tank, 1);  // Print temperature to 1 decimal place
  lcdTank.print(" C     ");  // Print the unit, then extra spaces to overwrite old digits from a longer previous reading

  // ---- HEX LCD ----
  lcdHEX.setCursor(8,0);  // Cursor is set to column 8 to start printing right after "HEX_IN: "
  lcdHEX.print(T_in,1);   // Print inlet temperature with 1 decimal place
  lcdHEX.print(" C   ");  // Unit plus padding spaces to erase leftover characters

  lcdHEX.setCursor(9,1);  // Cursor is set to column 9 to start printing right after "HEX_OUT: "
  lcdHEX.print(T_out,1);  // Print outlet temperature with 1 decimal place
  lcdHEX.print(" C   ");  // Unit plus padding spaces (only 7 columns remain on this row, so long values get cut off)

  // ---- SERIAL CSV LOGGING ----
  // Prints data to the Serial Monitor in a format Excel can read
  Serial.print(T_tank);   // Write the tank temperature (Serial.print defaults to 2 decimal places)
  Serial.print(",");      // Comma separates columns in a CSV file
  Serial.print(T_in);     // Write the inlet temperature
  Serial.print(",");      // Column separator
  Serial.println(T_out);  // Use println for the last value to start a new row

  delay(1000);   // 1-second sampling interval for transient thermal analysis (also paces the loop and LCD updates)
}
