// Handles the communication between Arduino and ProtoPie Connect

#include <Arduino.h>
#include <string.h>

// Declare struct
struct MessageValue {
  String message;
  String value;
};

// Store the latest message received
MessageValue receivedData;

// Function to parse the Message||Value format
MessageValue getMessage(String inputStr) {
  MessageValue result;
  char charArr[50];

  // Convert the Arduino String into a C-style character array for parsing
  inputStr.toCharArray(charArr, 50);

  // strtok looks for the first part of the string before "||"
  char* ptr = strtok(charArr, "||");

  if (ptr != NULL) {
    result.message = String(ptr); // Store the message part
    // Look for the part AFTER the "||"
    ptr = strtok(NULL, "||");
    if (ptr != NULL)
      result.value = String(ptr); // Store the value part
    else
      result.value = ""; // If no "||" was found, set value to empty string
  }
  return result; 
}

// Check Serial and update receivedData
void checkProtoPieMessages() {
  // Run if there is data waiting in the Serial buffer
  while (Serial.available() > 0) {
    // ProtoPie Connect 1.9.0+ uses the null character '\0' to end messages
    String receivedString = Serial.readStringUntil('\n');
    receivedString.trim();
    if (receivedString.length() > 0) {
      // Process the raw string and update our global receivedData object
      receivedData = getMessage(receivedString);
      
      // Debug: conferma cosa ha capito l'Arduino
      Serial.print("Received Message: ");
      Serial.println(receivedData.message);
    }
  }
}
