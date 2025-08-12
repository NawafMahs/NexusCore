# NexusCore
🚀 Enterprise-grade sensor monitoring and control system for industrial chocolate production machines. Built with .NET 9, EF Core, and ModBus RTU integration.
# Sensor System Documentation - DaireApplication

## Overview

This document provides comprehensive documentation for the sensor system implementation in the DaireApplication chocolate tempering machine control system. The system monitors and controls various sensors through ModBus RTU communication.

## Table of Contents

- [System Architecture](#system-architecture)
- [Sensor Types](#sensor-types)
- [Database Design](#database-design)
- [Implementation Guide](#implementation-guide)
- [API Usage](#api-usage)
- [ModBus Configuration](#modbus-configuration)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## System Architecture

```
┌─────────────────┐     ModBus RTU      ┌──────────────┐
│ Physical Sensors├────────────────────►│ PLC Controller│
└─────────────────┘                     └──────┬───────┘
                                               │
                                               ▼
┌─────────────────┐                     ┌──────────────┐
│ Application     │◄────────────────────┤ ModBus Service│
└────────┬────────┘                     └──────────────┘
         │
         ▼
┌─────────────────┐
│ SQL Database    │
└─────────────────┘
```

## Sensor Types

### Temperature Sensors (4 units)

| Sensor | Name | ModBus Address | Range | Purpose |
|--------|------|----------------|-------|---------|
| T-1 | Tank Bottom Temp | 8 | -14°C to 65°C | Monitor chocolate temperature at tank bottom |
| T-2 | Tank Wall Temp | 9 | -14°C to 65°C | Monitor chocolate temperature at tank wall |
| T-3 | Pump Temp | 10 | -10°C to 50°C | Monitor pump/circulation temperature |
| T-4 | Fountain Temp | 11 | -14°C to 65°C | Monitor dispensing temperature |

### Digital Sensors (3 units)

| Sensor | Name | ModBus Address | Type | Purpose |
|--------|------|----------------|------|---------|
| D-1 | Pedal | 1 (Bit 0) | Boolean | Manual dispensing control |
| D-2 | Cover Sensor | 1 (Bit 1) | Boolean | Safety interlock for cover |
| D-3 | E-Stop | 1 (Bit 2) | Boolean | Emergency stop button |

## Database Design

### Entity Relationship

```
Sensor (1) ────────────► (∞) SensorReading
   │                              │
   ├─ Id (PK)                    ├─ Id (PK)
   ├─ SensorType                 ├─ SensorId (FK)
   ├─ Name                       ├─ Timestamp
   ├─ ModBusAddress              ├─ Value
   └─ UnitType                   └─ IsValid
```

### Database Schema

```sql
-- Sensors table (static information)
CREATE TABLE Sensors (
    Id INT PRIMARY KEY IDENTITY(1,1),
    SensorType INT NOT NULL,
    Name NVARCHAR(100) NOT NULL,
    ModBusAddress INT NOT NULL,
    UnitType INT NOT NULL,
    IsActive BIT DEFAULT 1,
    CreatedAt DATETIME2 DEFAULT GETUTCDATE()
);

-- SensorReadings table (time-series data)
CREATE TABLE SensorReadings (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),
    SensorId INT NOT NULL,
    Timestamp DATETIME2 NOT NULL,
    Value DECIMAL(10,2) NOT NULL,
    IsValid BIT DEFAULT 1,
    FOREIGN KEY (SensorId) REFERENCES Sensors(Id)
);

-- Index for performance
CREATE INDEX IX_SensorReadings_SensorId_Timestamp 
ON SensorReadings(SensorId, Timestamp DESC);
```

## Implementation Guide

### 1. Enums and Constants

```csharp
// Sensor type enumeration
public enum SensorTypeEnum
{
    TankBottomTemp = 1,
    TankWallTemp = 2,
    PumpTemp = 3,
    FountainTemp = 4,
    Pedal = 5,
    CoverSensor = 6,
    EmergencyStop = 7
}

// Measurement unit types
public enum MeasurementUnitType
{
    Temperature = 1,
    Boolean = 2,
    Percentage = 3,
    Pressure = 4,
    Flow = 5
}

// Temperature units
public enum TemperatureUnit
{
    Celsius = 1,
    Fahrenheit = 2,
    Kelvin = 3
}

// Device states
public static class DeviceStatus
{
    public const string ON = "ON";
    public const string OFF = "OFF";
    public const string ON_AR = "تشغيل";
    public const string OFF_AR = "إيقاف";
    public const int STATE_OFF = 0;
    public const int STATE_ON = 1;
}
```

### 2. Value Objects

```csharp
// Temperature value object
public class Temperature : IComparable<Temperature>
{
    private readonly decimal _value;
    private readonly TemperatureUnit _unit;

    public Temperature(decimal value, TemperatureUnit unit = TemperatureUnit.Celsius)
    {
        _value = value;
        _unit = unit;
    }

    public decimal Value => _value;
    public TemperatureUnit Unit => _unit;
    
    public decimal ToCelsius()
    {
        return _unit switch
        {
            TemperatureUnit.Celsius => _value,
            TemperatureUnit.Fahrenheit => (_value - 32) * 5 / 9,
            TemperatureUnit.Kelvin => _value - 273.15m,
            _ => _value
        };
    }

    public static Temperature FromCelsius(decimal value) 
        => new Temperature(value, TemperatureUnit.Celsius);

    public override string ToString() => $"{_value}°C";
}

// Digital state value object
public class DigitalState
{
    public bool IsActive { get; }
    
    public DigitalState(bool isActive)
    {
        IsActive = isActive;
    }

    public static DigitalState On => new DigitalState(true);
    public static DigitalState Off => new DigitalState(false);
    
    public override string ToString() => IsActive ? "ON" : "OFF";
}
```

### 3. Entity Models

```csharp
// Sensor entity
public class Sensor
{
    public int Id { get; set; }
    public SensorTypeEnum SensorType { get; set; }
    public string Name { get; set; }
    public int ModBusAddress { get; set; }
    public MeasurementUnitType UnitType { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    // Navigation properties
    public ICollection<SensorReading> Readings { get; set; }
}

// Sensor reading entity
public class SensorReading
{
    public long Id { get; set; }
    public int SensorId { get; set; }
    public DateTime Timestamp { get; set; }
    public decimal Value { get; set; }
    public bool IsValid { get; set; } = true;
    
    // Navigation
    public Sensor Sensor { get; set; }
}
```

### 4. EF Core Configuration

```csharp
// Sensor configuration
public class SensorConfiguration : IEntityTypeConfiguration<Sensor>
{
    public void Configure(EntityTypeBuilder<Sensor> builder)
    {
        builder.ToTable("Sensors");
        
        builder.HasKey(s => s.Id);
        
        builder.Property(s => s.Name)
            .IsRequired()
            .HasMaxLength(100);
            
        builder.Property(s => s.SensorType)
            .HasConversion<int>();
            
        builder.Property(s => s.UnitType)
            .HasConversion<int>();
            
        builder.HasIndex(s => s.ModBusAddress)
            .IsUnique();
    }
}

// Sensor reading configuration
public class SensorReadingConfiguration : IEntityTypeConfiguration<SensorReading>
{
    public void Configure(EntityTypeBuilder<SensorReading> builder)
    {
        builder.ToTable("SensorReadings");
        
        builder.HasKey(r => r.Id);
        
        builder.Property(r => r.Value)
            .HasPrecision(10, 2);
            
        builder.HasIndex(r => new { r.SensorId, r.Timestamp })
            .IsDescending(false, true);
            
        builder.HasOne(r => r.Sensor)
            .WithMany(s => s.Readings)
            .HasForeignKey(r => r.SensorId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

## API Usage

### Service Implementation

```csharp
public interface ISensorService
{
    Task<Sensor> RegisterSensorAsync(string name, SensorTypeEnum type, int modbusAddress);
    Task<SensorReading> RecordReadingAsync(int sensorId, decimal value);
    Task<IEnumerable<SensorReading>> GetReadingsAsync(int sensorId, DateTime from, DateTime to);
    Task<SensorStatus> GetCurrentStatusAsync(int sensorId);
}

public class SensorService : ISensorService
{
    private readonly ApplicationDbContext _context;
    private readonly IModBusService _modbus;
    
    public SensorService(ApplicationDbContext context, IModBusService modbus)
    {
        _context = context;
        _modbus = modbus;
    }
    
    public async Task<SensorReading> RecordReadingAsync(int sensorId, decimal value)
    {
        var reading = new SensorReading
        {
            SensorId = sensorId,
            Value = value,
            Timestamp = DateTime.UtcNow,
            IsValid = true
        };
        
        _context.SensorReadings.Add(reading);
        await _context.SaveChangesAsync();
        
        return reading;
    }
    
    public async Task<SensorStatus> GetCurrentStatusAsync(int sensorId)
    {
        var sensor = await _context.Sensors
            .Include(s => s.Readings.OrderByDescending(r => r.Timestamp).Take(1))
            .FirstOrDefaultAsync(s => s.Id == sensorId);
            
        if (sensor == null)
            throw new NotFoundException($"Sensor {sensorId} not found");
            
        var latestReading = sensor.Readings.FirstOrDefault();
        
        return new SensorStatus
        {
            SensorId = sensor.Id,
            Name = sensor.Name,
            CurrentValue = latestReading?.Value ?? 0,
            Unit = GetUnitSymbol(sensor.UnitType),
            LastUpdate = latestReading?.Timestamp ?? DateTime.MinValue,
            IsOnline = latestReading?.Timestamp > DateTime.UtcNow.AddMinutes(-5)
        };
    }
}
```

### Controller Implementation

```csharp
[ApiController]
[Route("api/[controller]")]
public class SensorsController : ControllerBase
{
    private readonly ISensorService _sensorService;
    
    public SensorsController(ISensorService sensorService)
    {
        _sensorService = sensorService;
    }
    
    [HttpGet("{id}/status")]
    public async Task<ActionResult<SensorStatus>> GetStatus(int id)
    {
        var status = await _sensorService.GetCurrentStatusAsync(id);
        return Ok(status);
    }
    
    [HttpGet("{id}/readings")]
    public async Task<ActionResult<IEnumerable<SensorReading>>> GetReadings(
        int id, 
        [FromQuery] DateTime from, 
        [FromQuery] DateTime to)
    {
        var readings = await _sensorService.GetReadingsAsync(id, from, to);
        return Ok(readings);
    }
    
    [HttpPost("{id}/readings")]
    public async Task<ActionResult<SensorReading>> RecordReading(
        int id, 
        [FromBody] decimal value)
    {
        var reading = await _sensorService.RecordReadingAsync(id, value);
        return CreatedAtAction(nameof(GetStatus), new { id }, reading);
    }
}
```

## ModBus Configuration

### Connection Settings

```json
{
  "ModBusSettings": {
    "PortName": "COM1",
    "BaudRate": 115200,
    "DataBits": 8,
    "Parity": "None",
    "StopBits": 1,
    "SlaveAddress": 1,
    "Timeout": 1000
  }
}
```

### Reading Sensors via ModBus

```csharp
public class ModBusReader
{
    private readonly IModbusMaster _modbusMaster;
    
    public async Task<Dictionary<SensorTypeEnum, decimal>> ReadAllSensorsAsync()
    {
        var readings = new Dictionary<SensorTypeEnum, decimal>();
        
        // Read temperature sensors (Input Registers)
        var temps = await _modbusMaster.ReadInputRegistersAsync(1, 8, 4);
        readings[SensorTypeEnum.TankBottomTemp] = ConvertToTemperature(temps[0]);
        readings[SensorTypeEnum.TankWallTemp] = ConvertToTemperature(temps[1]);
        readings[SensorTypeEnum.PumpTemp] = ConvertToTemperature(temps[2]);
        readings[SensorTypeEnum.FountainTemp] = ConvertToTemperature(temps[3]);
        
        // Read digital sensors (Input Register bit fields)
        var digitals = await _modbusMaster.ReadInputRegistersAsync(1, 1, 1);
        readings[SensorTypeEnum.Pedal] = (digitals[0] & 0x01) > 0 ? 1 : 0;
        readings[SensorTypeEnum.CoverSensor] = (digitals[0] & 0x02) > 0 ? 1 : 0;
        readings[SensorTypeEnum.EmergencyStop] = (digitals[0] & 0x04) > 0 ? 1 : 0;
        
        return readings;
    }
    
    private decimal ConvertToTemperature(ushort rawValue)
    {
        // Convert from raw ModBus value to temperature
        // Assuming signed 16-bit with 0.1°C resolution
        short signedValue = (short)rawValue;
        return signedValue / 10.0m;
    }
}
```

## Best Practices

### 1. Data Retention Policy

```csharp
// Clean up old readings periodically
public async Task CleanupOldReadingsAsync(int daysToKeep = 30)
{
    var cutoffDate = DateTime.UtcNow.AddDays(-daysToKeep);
    
    await _context.SensorReadings
        .Where(r => r.Timestamp < cutoffDate)
        .ExecuteDeleteAsync();
}
```

### 2. Performance Optimization

```csharp
// Use bulk insert for multiple readings
public async Task RecordBulkReadingsAsync(
    Dictionary<int, decimal> sensorValues)
{
    var readings = sensorValues.Select(kv => new SensorReading
    {
        SensorId = kv.Key,
        Value = kv.Value,
        Timestamp = DateTime.UtcNow
    }).ToList();
    
    await _context.SensorReadings.AddRangeAsync(readings);
    await _context.SaveChangesAsync();
}
```

### 3. Error Handling

```csharp
public class SensorReadingValidator
{
    private readonly Dictionary<SensorTypeEnum, (decimal min, decimal max)> _ranges = new()
    {
        { SensorTypeEnum.TankBottomTemp, (-14, 65) },
        { SensorTypeEnum.TankWallTemp, (-14, 65) },
        { SensorTypeEnum.PumpTemp, (-10, 50) },
        { SensorTypeEnum.FountainTemp, (-14, 65) }
    };
    
    public bool ValidateReading(SensorTypeEnum type, decimal value)
    {
        if (_ranges.TryGetValue(type, out var range))
        {
            return value >= range.min && value <= range.max;
        }
        
        return true; // No validation for digital sensors
    }
}
```

## Troubleshooting

### Common Issues

1. **ModBus Communication Errors**
   - Check cable connections
   - Verify baud rate settings
   - Ensure correct slave address

2. **Invalid Temperature Readings**
   - Verify sensor wiring
   - Check for electromagnetic interference
   - Calibrate sensors if needed

3. **Database Performance**
   - Ensure indexes are created
   - Implement data retention policy
   - Consider partitioning for large datasets

### Debug Logging

```csharp
public class SensorLogger
{
    private readonly ILogger<SensorLogger> _logger;
    
    public void LogReading(int sensorId, decimal value, bool isValid)
    {
        if (!isValid)
        {
            _logger.LogWarning(
                "Invalid reading from sensor {SensorId}: {Value}", 
                sensorId, value);
        }
        else
        {
            _logger.LogDebug(
                "Sensor {SensorId} reading: {Value}", 
                sensorId, value);
        }
    }
}
```

## Testing

### Unit Test Example

```csharp
[TestClass]
public class SensorServiceTests
{
    [TestMethod]
    public async Task RecordReading_ValidTemperature_SavesSuccessfully()
    {
        // Arrange
        var context = GetInMemoryContext();
        var service = new SensorService(context, null);
        var sensor = new Sensor 
        { 
            Id = 1, 
            SensorType = SensorTypeEnum.TankBottomTemp,
            UnitType = MeasurementUnitType.Temperature 
        };
        context.Sensors.Add(sensor);
        await context.SaveChangesAsync();
        
        // Act
        var reading = await service.RecordReadingAsync(1, 45.5m);
        
        // Assert
        Assert.IsNotNull(reading);
        Assert.AreEqual(45.5m, reading.Value);
        Assert.AreEqual(1, await context.SensorReadings.CountAsync());
    }
}
```

## Seed Data

```csharp
public static class SensorSeedData
{
    public static void Seed(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Sensor>().HasData(
            new Sensor { Id = 1, Name = "Tank Bottom Temperature", 
                        SensorType = SensorTypeEnum.TankBottomTemp, 
                        ModBusAddress = 8, 
                        UnitType = MeasurementUnitType.Temperature },
            new Sensor { Id = 2, Name = "Tank Wall Temperature", 
                        SensorType = SensorTypeEnum.TankWallTemp, 
                        ModBusAddress = 9, 
                        UnitType = MeasurementUnitType.Temperature },
            new Sensor { Id = 3, Name = "Pump Temperature", 
                        SensorType = SensorTypeEnum.PumpTemp, 
                        ModBusAddress = 10, 
                        UnitType = MeasurementUnitType.Temperature },
            new Sensor { Id = 4, Name = "Fountain Temperature", 
                        SensorType = SensorTypeEnum.FountainTemp, 
                        ModBusAddress = 11, 
                        UnitType = MeasurementUnitType.Temperature },
            new Sensor { Id = 5, Name = "Pedal Sensor", 
                        SensorType = SensorTypeEnum.Pedal, 
                        ModBusAddress = 1, 
                        UnitType = MeasurementUnitType.Boolean },
            new Sensor { Id = 6, Name = "Cover Sensor", 
                        SensorType = SensorTypeEnum.CoverSensor, 
                        ModBusAddress = 1, 
                        UnitType = MeasurementUnitType.Boolean },
            new Sensor { Id = 7, Name = "Emergency Stop", 
                        SensorType = SensorTypeEnum.EmergencyStop, 
                        ModBusAddress = 1, 
                        UnitType = MeasurementUnitType.Boolean }
        );
    }
}
```

## Contributing

When adding new sensors:

1. Add the sensor type to `SensorTypeEnum`
2. Update the seed data
3. Add validation rules if applicable
4. Update ModBus reading logic
5. Add unit tests
6. Update this documentation

## License

This sensor system is part of the DaireApplication project.

---

**Last Updated**: January 2024  
**Version**: 1.0.0  
**Author**: DaireApplication Development Team
