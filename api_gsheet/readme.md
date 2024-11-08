
# API Google Sheet para Connected Content en Braze

Creación de API automática para actualización de contenidos automaticos en los flujos de comunicación de la herramienta Braze a través de la funcionalidad Connected Content.


![Logo](https://cdn.rubyonremote.com/ctbroz6tre84wcrfh354x20i76ai) 
## Estructura Gsheet

| Country | Beneficio  | Monto  |
|---------|------------|--------|
| CO      | Bienvenida | 400€   |
| ES      | Bienvenida | 3.000€ |
| AR      | Bienvenida | 500€   |
| UY      | Bienvenida | 3.000€ |
| CL      | Bienvenida | 400€   |


# Estructura Connected Content Braze

[Connected Content Braze Documentation](https://www.braze.com/docs/user_guide/personalization_and_dynamic_content/connected_content/local_connected_content_variables/#json-parsing)

```ruby
{% connected_content https://example.com/api/endpoint :save result %}
{% if result.__http_status_code__ != 200 %}
{% abort_message('Connected Content returned a non-200 status code') %}
{% endif %}
```

## Implementar la API del Google App Script como Aplicación Web (No segura)

![App Screenshot](https://i.postimg.cc/MG5fV8nK/sheetmonlab1.png)


## Postman Test

![App Screenshot](https://i.postimg.cc/vZL4wdLL/get-response-juan.png)

##  Sheetlabs Response Example

![App Screenshot](https://i.postimg.cc/jqLZXRy1/sheetlabs-response-ex.png)

##  Post Example

![App Screenshot](https://i.postimg.cc/dtgXdLXn/post-example-juan.png)

## Resultado Gsheet

| Country       | Beneficio  | Monto  |
|---------------|------------|--------|
| CO            | Bienvenida | 400€   |
| ES            | Bienvenida | 3.000€ |
| AR            | Bienvenida | 500€   |
| UY            | Bienvenida | 3.000€ |
| CL            | Bienvenida | 400€   |
| Hola mundo    |            |        |

## API Get generic N col

```js
// Función para verificar el nombre de la hoja y devolver la hoja si existe
function getSheetByName(sheetName) {
  if (!sheetName) {
    return { success: false, message: "No se ha proporcionado el nombre de la hoja." };
  }

  var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(sheetName);
  if (!sheet) {
    return { success: false, message: "La hoja '" + sheetName + "' no existe." };
  }

  return { success: true, sheet: sheet };
}

// Función para crear respuestas de error
function createErrorResponse(message) {
  return ContentService.createTextOutput("Error: " + message)
    .setMimeType(ContentService.MimeType.TEXT);
}

// Función para validar JSON
function validateJson(jsonData, requiredFields) {
  for (var field of requiredFields) {
    if (!jsonData[field]) {
      return `Falta el campo '${field}' en el JSON.`;
    }
  }
  return null;
}

function doGet(e) {
  var sheetName = e.parameter.sheetName;
  var sheetResult = getSheetByName(sheetName);

  if (!sheetResult.success) {
    return createErrorResponse(sheetResult.message);
  }

  var sheet = sheetResult.sheet;
  var range = sheet.getDataRange();
  var values = range.getValues();

  if (values.length < 2) {
    return createErrorResponse("La hoja no tiene suficientes datos.");
  }

  var headers = values[0];
  var jsonData = [];
  var filters = {};

  for (var param in e.parameter) {
    if (param !== "sheetName" && headers.includes(param)) {
      filters[param] = e.parameter[param];
    }
  }

  for (var i = 1; i < values.length; i++) {
    var rowData = {};
    var match = true;

    for (var j = 0; j < headers.length; j++) {
      rowData[headers[j]] = values[i][j];
    }

    for (var column in filters) {
      if (rowData[column] != filters[column]) {
        match = false;
        break;
      }
    }

    if (match) {
      jsonData.push(rowData);
    }
  }

  return ContentService.createTextOutput(JSON.stringify(jsonData))
    .setMimeType(ContentService.MimeType.JSON);
}

function doPost(e) {
  var method = e.parameter._method ? e.parameter._method.toUpperCase() : "POST";
  var sheetName = e.parameter.sheetName;
  var sheetResult = getSheetByName(sheetName);

  if (!sheetResult.success) {
    return createErrorResponse(sheetResult.message);
  }

  var sheet = sheetResult.sheet;

  if (method === "PUT") {
    return handlePut(e, sheet);
  } else if (method === "PATCH") {
    return handlePatch(e, sheet);
  } else if (method === "DELETE") {
    return handleDelete(e, sheet);
  } else {
    return handlePost(e, sheet);
  }
}

// Función para manejar el método POST normal
function handlePost(e, sheet) {
  var requestBody = e.postData.contents;
  var jsonData;

  try {
    jsonData = JSON.parse(requestBody); 
  } catch (error) {
    return createErrorResponse("El cuerpo de la solicitud no es un JSON válido.");
  }

  if (!jsonData.updates || !Array.isArray(jsonData.updates)) {
    return createErrorResponse("El cuerpo JSON debe contener un array 'updates'.");
  }

  var errores = [];
  var dataToSet = [];

  jsonData.updates.forEach(function(update) {
    var row = update.row;
    var values = update.values;
    if (row && values && Array.isArray(values)) {
      dataToSet.push({ row, values });
    }
  });

  dataToSet.sort((a, b) => a.row - b.row);

  try {
    dataToSet.forEach(function(item) {
      var range = sheet.getRange(item.row, 1, 1, item.values.length);
      range.setValues([item.values]);
    });
  } catch (error) {
    return createErrorResponse("Error al actualizar las filas en batch: " + error.message);
  }

  return ContentService.createTextOutput("Actualización exitosa de las filas.")
    .setMimeType(ContentService.MimeType.TEXT);
}

// Función para manejar el método PUT
function handlePut(e, sheet) {
  var requestBody = e.postData.contents;
  var jsonData;

  try {
    jsonData = JSON.parse(requestBody);
  } catch (error) {
    return createErrorResponse("El cuerpo de la solicitud no es un JSON válido.");
  }

  var validationError = validateJson(jsonData, ['row', 'values']);
  if (validationError) {
    return createErrorResponse(validationError);
  }

  var row = jsonData.row;
  var values = jsonData.values;

  try {
    var range = sheet.getRange(row, 1, 1, values.length);
    range.setValues([values]); 
  } catch (error) {
    return createErrorResponse("Error al actualizar la fila: " + error.message);
  }

  return ContentService.createTextOutput("Fila " + row + " reemplazada correctamente.")
    .setMimeType(ContentService.MimeType.TEXT);
}

// Función para manejar el método PATCH
function handlePatch(e, sheet) {
  var requestBody = e.postData.contents;
  var jsonData;

  try {
    jsonData = JSON.parse(requestBody);
  } catch (error) {
    return createErrorResponse("El cuerpo de la solicitud no es un JSON válido.");
  }

  var validationError = validateJson(jsonData, ['row', 'updates']);
  if (validationError) {
    return createErrorResponse(validationError);
  }

  var row = jsonData.row;
  var updates = jsonData.updates;
  var errores = [];

  updates.forEach(function(update) {
    var column = update.column;
    var value = update.value;

    if (!column || value === undefined) {
      errores.push("Faltan 'column' o 'value' en la actualización.");
      return;
    }

    try {
      sheet.getRange(row, column).setValue(value);
    } catch (error) {
      errores.push("Error al actualizar columna " + column + " en la fila " + row + ": " + error.message);
    }
  });

  if (errores.length > 0) {
    return createErrorResponse(errores.join(", "));
  }

  return ContentService.createTextOutput("Fila " + row + " actualizada parcialmente.")
    .setMimeType(ContentService.MimeType.TEXT);
}

// Función para manejar el método DELETE
function handleDelete(e, sheet) {
  var requestBody = e.postData.contents;
  var jsonData;

  try {
    jsonData = JSON.parse(requestBody);
  } catch (error) {
    return createErrorResponse("El cuerpo de la solicitud no es un JSON válido.");
  }

  var validationError = validateJson(jsonData, ['rows']);
  if (validationError) {
    return createErrorResponse(validationError);
  }

  var rows = jsonData.rows;
  var errores = [];

  rows.sort((a, b) => b - a); 

  rows.forEach(function(row) {
    try {
      sheet.deleteRow(row);
    } catch (error) {
      errores.push("Error al eliminar la fila " + row + ": " + error.message);
    }
  });

  if (errores.length > 0) {
    return createErrorResponse(errores.join(", "));
  }

  return ContentService.createTextOutput("Filas eliminadas correctamente.")
    .setMimeType(ContentService.MimeType.TEXT);
}

```
## Get example Postman

![App Screenshot](https://i.postimg.cc/Hktcq7Qk/n-columns.png)