---
layout: post
title: Revisiting how to teach Object-oriented programming
---

This article is co-authored with [Alexis Cao](https://twitter.com/acaopro), with whom we often discuss programming.
After a few years, we came into the conclusion that object-oriented programming (OOP) might be taught wrongly, 
especially in programming schools.

# What is Object-oriented programming ?

Quoting Wikipedia:

> Object-oriented programming (OOP) is a programming paradigm based on the concept of "objects", which can contain data 
> and code: data in the form of fields (often known as attributes or properties), and code, in the form of procedures 
> (often known as methods). 

OOP is also often tied with 4 main principles.

- **Encapsulation**
- **Abstraction**
- **Inheritance**
- **Polymorphism**

Let's illustrate them as a programming school would, and with the canonical class-based OOP language: Java.

## Encapsulation

Object can be used to store a state using one or more attributes. To prevent storing inconsistent data, we can make them
private and expose public methods to mutate them.

```java
class BadCelsiusTemperature {
  public double value;
}

var badTemp = new BadCelsiusTemperature();
badTemp.value = -300; // Invalid state: temperature cannot go below -273.15 °C

class CelsiusTemperature {
  private double value;
    
  public CelsiusTemperature(double value) {
    setValue(value);
  }
    
  public double getValue() {
    return value;
  }
    
  public void setValue(double value) {
    // This public setter enforce state consistency
    if (value < -273.15) {
      throw new IllegalArgumentException("Invalid temperature");
    }
    this.value = value;
  }
}

var temp = new CelsiusTemperature(10); // This is allowd
var invalidTemp = new CelsiusTemperature(-300); // This will throw an exception
```

## Abstraction

The goal of abstraction is to be able to express ideas without attaching a concrete object behind. This decouples an
idea from its implementation.

```java
// This interface express the concept of a device that measures the temperature, without
// expressing how it will be done
interface Thermometer {
  CelsiusTemperature readTemperature();
}
```

## Inheritance

Inheritance expresses an is-a relationship between objects. A children class can inherit from parent classes' behavior,
so common behavior can then be factorized.

```java
// This abstract class describe a kind of device that needs to be initialized
// before each temperature reading and can read temperature in Kelvin
abstract class ThermometerDevice implements Thermometer {
  @Override
  public CelsiusTemperature readTemperature() {
    initializeRead();
    
    // Convert the temperature
    double kelvinTemp = readTemperatureInKelvin();
    return new CelsiusTemperature(kelvinTemp - 273.15);
  }
  
  protected void initializeRead();

  protected double readTemperatureInKelvin();
}

// This class uses a "ElectronicThermometerConnector" to
// connect to the electronic thermometer.
// It inherits from the abstract ThermometerDevice for
// common behavior (initializing the reads and temperature conversion) 
// and only implement specific bits that deals with the connector.
class ElectronicThermometerDevice extends ThermometerDevice {
  private ElectronicThermometerConnector connector;

  public ElectronicThermometerDevice() {
    connector = new ElectronicThermometerConnector();
  }
  
  @Override
  protected void initializeRead() {
    connector.waitForSensorSync();
  }

  @Override
  protected double readTemperatureInKelvin() {
    return connector.read();
  }
}
```

## Polymorphism

Closely related to the two previous principles, polymorphism states that a behavior can be implemented using only
abstractions, and used with concrete classes. Generic behaviors can then be implemented easily.

```java
// Implementation of the algorithm uses an interface
CelsiusTemperature getAverageTemperature(Thermometer thermometer) {
  double temp = 0;
  for (int i = 0; i < 100; ++i) {
    var celsiusTemp = thermometer.readTemperature();
    temp = temperature + celsiusTemp.getValue();
  }
  return new CelsiusTemperature(temp / 100);
}

// When using the algorithm, concrete classes are used instead
var averageTemp = getAverageTemperature(new ElectronicThermometerDevice());
var fakeTemp = getAverageTemperature(new FakeThermometer());
```

# 