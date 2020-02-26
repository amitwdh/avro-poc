# Avro Serialization POC

## What is Avro?
Avro is data serialization system developed within Apache's Hadoop project. It uses JSON for defining data types and protocols, and serializes data in a compact binary format.

## Apache Avro Maven Dependency
```xml
<dependency>
  <groupId>org.apache.avro</groupId>
  <artifactId>avro</artifactId>
  <version>1.9.1</version>
</dependency>
```
## Avro Maven Plugin
```xml
<plugin>
  <groupId>org.apache.avro</groupId>
  <artifactId>avro-maven-plugin</artifactId>
  <version>${avro.version}</version>
  <executions>
    <execution>
      <phase>generate-sources</phase>
      <goals>
        <goal>schema</goal>
      </goals>
      <configuration>
        <sourceDirectory>${project.basedir}/src/main/resources/</sourceDirectory>
        <outputDirectory>${project.basedir}/src/main/java/</outputDirectory>
        <includes>
          <include>**/*.avsc</include>
        </includes>
        <stringType>String</stringType>
      </configuration>
    </execution>
  </executions>
</plugin>
```
## Avro Serializer
```java
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.util.Map;
import javax.xml.bind.DatatypeConverter;
import org.apache.avro.generic.GenericRecord;
import org.apache.avro.io.BinaryEncoder;
import org.apache.avro.io.DatumWriter;
import org.apache.avro.io.EncoderFactory;
import org.apache.avro.specific.SpecificDatumWriter;
import org.apache.avro.specific.SpecificRecordBase;
import org.apache.kafka.common.errors.SerializationException;
import org.apache.kafka.common.serialization.Serializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
 
public class AvroSerializer<T extends SpecificRecordBase> implements Serializer<T> {
 
    private static final Logger LOGGER = LoggerFactory.getLogger(AvroSerializer.class);
 
    @Override
    public void close() {
    }
 
    @Override
    public void configure(Map<String, ?> arg0, boolean arg1) {
    }
 
    @Override
    public byte[] serialize(String topic, T data) {
        try {
            byte[] result = null;
 
            if (data != null) {
                LOGGER.debug("data='{}'", data);
 
                ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
                BinaryEncoder binaryEncoder = EncoderFactory.get().binaryEncoder(byteArrayOutputStream, null);
 
                DatumWriter<GenericRecord> datumWriter = new SpecificDatumWriter<>(data.getSchema());
                datumWriter.write(data, binaryEncoder);
 
                binaryEncoder.flush();
                byteArrayOutputStream.close();
 
                result = byteArrayOutputStream.toByteArray();
                LOGGER.debug("serialized data='{}'", DatatypeConverter.printHexBinary(result));
            }
            return result;
        } catch (IOException ex) {
            throw new SerializationException("Can't serialize data='" + data + "' for topic='" + topic + "'", ex);
        }
    }
}
```

## Avro Deserializer
```java
import java.util.Arrays;
import java.util.Map;
import javax.xml.bind.DatatypeConverter;
import org.apache.avro.generic.GenericRecord;
import org.apache.avro.io.DatumReader;
import org.apache.avro.io.Decoder;
import org.apache.avro.io.DecoderFactory;
import org.apache.avro.specific.SpecificDatumReader;
import org.apache.avro.specific.SpecificRecordBase;
import org.apache.kafka.common.errors.SerializationException;
import org.apache.kafka.common.serialization.Deserializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import com.walmart.replenishment.odr.constant.Constants;
 
public class AvroDeserializer<T extends SpecificRecordBase> implements Deserializer<T> {
 
    private static final Logger LOGGER = LoggerFactory.getLogger(AvroDeserializer.class);
 
    protected Class<T> targetType = null;
 
    public AvroDeserializer() {
    }
 
    @Override
    public void close() {
    }
 
    @SuppressWarnings("unchecked")
    @Override
    public void configure(Map<String, ?> arg0, boolean arg1) {
        try {
            targetType = (Class<T>) Class.forName((String) arg0.get(Constants.AVRO_MSG_CLASS_TYPE));
            LOGGER.info("AvroDeserializer configured successfully!");
        } catch (ClassNotFoundException e) {
            LOGGER.error("Class Not Found " + Constants.AVRO_MSG_CLASS_TYPE, e);
        }
    }
 
    @SuppressWarnings("unchecked")
    @Override
    public T deserialize(String topic, byte[] data) {
        try {
            T result = null;
 
            if (data != null) {
                LOGGER.debug("data='{}'", DatatypeConverter.printHexBinary(data));
 
                DatumReader<GenericRecord> datumReader = new SpecificDatumReader<GenericRecord>(
                        targetType.newInstance().getSchema());
                Decoder decoder = DecoderFactory.get().binaryDecoder(data, null);
 
                result = (T) datumReader.read(null, decoder);
                ;
                LOGGER.debug("deserialized data='{}'", result);
            }
            return result;
        } catch (Exception ex) {
            throw new SerializationException(
                    "Can't deserialize data '" + Arrays.toString(data) + "' from topic '" + topic + "'", ex);
        }
    }
}
```
## Avro Data Types
Please refer Avro 1.9.1 Official Documentation for details https://avro.apache.org/docs/current/spec.html#preamble

Primitive type: null, boolean, int, long, float, double, bytes, string.

Complex types: records, enums, arrays, maps, unions and fixed.

Logical Types: Decimal, UUID, Date, Time (millisecond precision), Time (microsecond precision), Timestamp (millisecond precision), Timestamp (microsecond precision), Duration.

Note: Applications need only add <stringType>String</stringType> to the configuration of avro-maven-plugin in their pom.xml to switch to using String in their generated code. Be default it generates CharSequence.

## Avro Record Schema
- Avro Record Schema defined using JSON 
- It has some common fields:
**	Name:** Name of your schema
**	Namespace:** (equivalent to package in java)
**	Doc:** Documentation to explain your schema
**	Aliases:** Optional Other name for your schema
	**Fields:**
** 		Name:** Name of your field
**		Doc:** Documentation for that field
**		Type:** Data Type for that field (can be primitive type)
**		Default:** Default value for that field
- Example 1, let's say we have Customer class with below fields,
First name, Last Name, Age, Height in cm, Weight in kg, Automated Email turn on (boolean, default true)
###### Customer Avro Schema
```json
{
  "type": "record",
  "namespace": "com.example",
  "name": "Customer ",
  "doc": "Avro Schema for Customer",
  "fields": [
    {
      "name": "first_name",
      "type": "string",
      "doc": "First name of Customer"
    },
    {
      "name": "last_name",
      "type": "string",
      "doc": "Last name of Customer"
    },
    {
      "name": "age",
      "type": "int",
      "doc": "Age of Customer"
    },
    {
      "name": "height",
      "type": "float",
      "doc": "Height in cm"
    },
    {
      "name": "weight",
      "type": "float",
      "doc": "Weight in cm"
    },
    {
      "name": "automated_email",
      "type": "boolean",
      "default": true,
      "doc": "true if user want marketing email"
    }
  ]
}
```
- Example 2, let's say we have Customer class with below fields,
First name, Last Name, Age, Height in cm, Weight in kg, Automated Email turn on (boolean, default true), Customer email ids, Customer Address (address, city, postcode, type), sent time (timestamp)
```json
[
  {
    "type": "record",
    "namespace": "com.example",
    "name": "CustomerAddress",
    "fields": [
      {
        "name": "address",
        "type": "string"
      },
      {
        "name": "city",
        "type": "string"
      },
      {
        "name": "postcode",
        "type": [
          "int",
          "string"
        ],
        "doc": "postcode can either be String or Int"
      },
      {
        "name": "type",
        "type": [
          {
            "type": "enum",
            "name": "AddressType",
            "symbols": [
              "PO_BOX",
              "ENTERPRISE",
              "RESIDENTIAL"
            ]
          }
        ]
      }
    ]
  },
  {
    "type": "record",
    "namespace": "com.example",
    "name": "Customer",
    "doc": "Avro Schema for Customer",
    "fields": [
      {
        "name": "first_name",
        "type": "string",
        "doc": "First name of Customer"
      },
      {
        "name": "last_name",
        "type": [
          "string",
          "null"
        ],
        "default": "null",
        "doc": "Last name of Customer"
      },
      {
        "name": "age",
        "type": "int",
        "doc": "Age of Customer"
      },
      {
        "name": "height",
        "type": "float",
        "doc": "Height in cm"
      },
      {
        "name": "weight",
        "type": "float",
        "doc": "Weight in cm"
      },
      {
        "name": "automated_email",
        "type": "boolean",
        "default": true,
        "doc": "true if user want marketing email"
      },
      {
        "name": "customer_emails",
        "type": {
          "type": "array",
          "items": "string",
          "default": []
        }
      },
      {
        "name": "customer_address",
        "type": "com.example.CustomerAddress"
      },
      {
        "name": "sent_time",
        "type": {
          "type": "long",
          "logicalType": "timestamp-millis"
        }
      }
    ]
  }
]
```
## Enum
Schema syntax given on Avro official documentation doesn't work. Correct way to create example enum is as below,
```json
{
        "name": "type",
        "type": [
          {
            "type": "enum",
            "name": "AddressType",
            "symbols": [
              "PO_BOX",
              "ENTERPRISE",
              "RESIDENTIAL"
            ]
          }
        ]
      }
```

## Array
Schema syntax given on Avro official documentation doesn't work. Correct way to create example array is as below,
```json
{
        "name": "customer_emails",
        "type": {
          "type": "array",
          "items": "string",
          "default": []
        }
      }
```
Example to demonstrate List<Child> customer_email; in avro schema,
```json
{
  "name": "customer_emails",
  "type": {
    "type": "array",
    "items": {
      "type": "record",
      "name": "Child",
      "fields": [
        {
          "name": "name",
          "type": "string"
        }
      ]
    }
  }
}
```
## Array Of Custom Object
```json
{
  "name": "customer_emails",
  "type": {
    "type": "array",
    "items": {
      "type": "record",
      "name": "Child",
      "fields": [
        {
          "name": "name",
          "type": "string"
        }
      ]
    }
  }
}
```

## Timestamp
Generated java class will be having Instant type for send_time variable. I haven't find anywhere any way to get direct timestamp object. We can convert Instant to Timestamp in java layer.
```json
{
        "name": "sent_time",
        "type": {
          "type": "long",
          "logicalType": "timestamp-millis"
        }
      }
```

## UUID Logical Type
Generated java class will be having String type for uuid logical type variable. We can convert String to UUID.
```json
{
        "name": "uuid",
        "type": {
          "type": "string",
          "logicalType": "uuid"
        }
      }
```

## Duration
According to Official document, avro supports Duration data type but it doesn't generate proper java classes to set duration object.
```json
{
        "name": "duration",
        "type": {
          "type": "fixed",
          "size": 12,
          "logicalType": "duration"
        }
      }
```
instead of using above way we can convert Duration to long and use long data type in schema as below,
```json
{
  "name": "duration",
  "type": "long"
}
```

## Map
Correct syntax to define map in schema is as shown in below example, Map keys are assumed to be strings. Key's can not be other than String type. 
```json
{
  "name": "nameMap",
  "type": {
    "type": "map",
    "values": "string"
   }
}
```
Example to demonstrate Map<String, Map<String, String>> as below,
```json
{
  "name": "nestedMap",
  "type": {
    "type": "map",
    "values": {
      "type": "map",
      "values": "string"
    }
  }
}
```
Example to demonstrate Map<String, UserDefinedObject> as below,
```json
[
  {
    "type": "record",
    "namespace": "com.example",
    "name": "AvroZoneDateTime",
    "fields": [
      {
        "name": "zoneId",
        "type": "string"
      },
      {
        "name": "timestamp",
        "type": {
          "type": "long",
          "logicalType": "timestamp-millis"
        }
      }
    ]
  },
  {
    "type": "record",
    "namespace": "com.example",
    "name": "CustomerAddress",
    "fields": [
      {
        "name": "mapStringToCustomObject",
        "type": {
          "type": "map",
          "values": {
            "type": "com.example.AvroZoneDateTime"
          }
        }
      }
    ]
  }
]
```
## ZoneDateTime
Avro doesn't support ZoneDateTime, we should create another record which will have Timestamp and ZoneId and can re-generate ZoneDateTime object using those fields. Schema definition will something like below,
```json
[
  {
    "type": "record",
    "namespace": "com.example",
    "name": "AvroZoneDateTime",
    "fields": [
      {
        "name": "zoneId",
        "type": "string"
      },
      {
        "name": "sent_time",
        "type": {
          "type": "long",
          "logicalType": "timestamp-millis"
        }
      }
    ]
  },
  {
    "type": "record",
    "namespace": "com.example",
    "name": "Customer",
    "doc": "Avro Schema for Customer",
    "fields": [
      {
        "name": "zoneDateTime",
        "type": "com.example.AvroZoneDateTime"
      }
    ]
  }
]
```
## Date
The date logical type represents a date within the calendar, with no reference to a particular time zone or time of day. A date logical type annotates an Avro int, where the int stores the number of days from the unix epoch, 1 January 1970 (ISO calendar).
```json
[
  {
    "type": "record",
    "namespace": "com.example",
    "name": "CustomerAddress",
    "fields": [
      {
        "name": "loginDate",
        "type": {
          "type": "int",
          "logicalType": "date"
        }
      }
    ]
  }
]
```
Above schema generate LocalDate type java class variable.

## Generating Avro Schema from POJO definition
Ok but wait -- you do not have to START with an Avro Schema. This module can actually generate schema for you, starting with POJO definition(s)! Here's how
```json
public class POJO {
  // your typical, Jackson-compatible POJO (with or without annotations)
}

ObjectMapper mapper = new ObjectMapper(new AvroFactory());
AvroSchemaGenerator gen = new AvroSchemaGenerator();
mapper.acceptJsonFormatVisitor(RootType.class, gen);
AvroSchema schemaWrapper = gen.getGeneratedSchema();

org.apache.avro.Schema avroSchema = schemaWrapper.getAvroSchema();
String asJson = avroSchema.toString(true);
```
So: you can generate native Avro Schema object very easily, and use that instead of hand-crafted variant. Or you can even use this method for outputting schemas to use in other processing systems; use your POJOs as origin of schemata.

## Official Documentation
https://avro.apache.org/docs/current/gettingstartedjava.html
https://avro.apache.org/docs/current/spec.html#preamble

## Avro Version
Apache Avro 1.9.1

###End
