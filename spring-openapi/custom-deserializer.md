
## **1. Create a Generic Enum Deserializer**

We can create a generic deserializer that works for any enum type. Here's how:

```java
package hu.vereba.cm.deserializer;

import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.databind.deser.ContextualDeserializer;
import hu.vereba.cm.exception.InvalidEnumValueException;

import java.io.IOException;

public class GenericEnumDeserializer extends JsonDeserializer<Enum<?>> implements ContextualDeserializer {

    private Class<Enum<?>> enumType;

    public GenericEnumDeserializer() {
        // Default constructor
    }

    public GenericEnumDeserializer(Class<Enum<?>> enumType) {
        this.enumType = enumType;
    }

    @Override
    public Enum<?> deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JsonProcessingException {
        String value = p.getValueAsString();

        if (enumType == null) {
            // Obtain the enum type from the context
            enumType = (Class<Enum<?>>) ctxt.getContextualType().getRawClass();
        }

        for (Enum<?> enumConstant : enumType.getEnumConstants()) {
            // Assuming the enum has a getValue() method; adjust if necessary
            if (enumConstant.toString().equals(value)) {
                return enumConstant;
            }
        }

        // Get the field name causing the error
        String fieldName = p.getCurrentName();

        // Throw custom exception with field name and invalid value
        throw new InvalidEnumValueException(p, fieldName, value);
    }

    @Override
    public JsonDeserializer<?> createContextual(DeserializationContext ctxt, BeanProperty property) throws JsonMappingException {
        // Obtain the enum type from the property
        JavaType type = property.getType();
        Class<?> rawClass = type.getRawClass();

        return new GenericEnumDeserializer((Class<Enum<?>>) rawClass);
    }
}
```

## **2. Modify Your Enums to Use the Generic Deserializer**

Annotate your enums with the `@JsonDeserialize` annotation to use the `GenericEnumDeserializer`.

Add the following configuration to your `pom.xml` to automatically annotate all enums:

```xml
<additionalEnumTypeAnnotations>
    @com.fasterxml.jackson.databind.annotation.JsonDeserialize(using = hu.vereba.cm.deserializer.GenericEnumDeserializer.class)
</additionalEnumTypeAnnotations>
```

## **3. Create the `InvalidEnumValueException` Class**

```java
package hu.vereba.cm.exception;

import com.fasterxml.jackson.databind.JsonMappingException;
import com.fasterxml.jackson.core.JsonParser;

public class InvalidEnumValueException extends JsonMappingException {

    private final String fieldName;
    private final String invalidValue;

    public InvalidEnumValueException(JsonParser parser, String fieldName, String invalidValue) {
        super(parser, "Invalid value '" + invalidValue + "' for field '" + fieldName + "'");
        this.fieldName = fieldName;
        this.invalidValue = invalidValue;
    }

    public String getFieldName() {
        return fieldName;
    }

    public String getInvalidValue() {
        return invalidValue;
    }

    @Override
    public String getMessage() {
        return "Invalid value '" + invalidValue + "' for field '" + fieldName + "'";
    }
}
```

---

## **4. Implement a Global Exception Handler**

```java
package hu.vereba.cm.exception;

import hu.vereba.cm.rest.model.ErrorMessage;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(InvalidEnumValueException.class)
    public ResponseEntity<ErrorMessage> handleInvalidEnumValueException(InvalidEnumValueException ex) {
        ErrorMessage errorMessage = new ErrorMessage();
        errorMessage.setError("INVALID_ENUM_VALUE");
        errorMessage.setDetails("Invalid value '" + ex.getInvalidValue() + "' for field '" + ex.getFieldName() + "'");
        return new ResponseEntity<>(errorMessage, HttpStatus.BAD_REQUEST);
    }

    // Handle other exceptions as needed
}
```

