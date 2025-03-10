Problem Statement

Context: The company I worked for specializes in managing promotional services in stores, mainly supermarkets. The promotional staff is responsible for:

Promoting a specific brand's product line.

Arranging products to make them more visually appealing on shelves.

Negotiating with store management to gain more shelf space for our clients' brands.

Installing temporary and permanent displays to make products more attractive to customers.

KPI Measured: One of the key performance indicators (KPIs) the client uses to evaluate and fully pay for the service is service hours.

Control System: Our field staff use an app on their phones provided by the agency for check-in and check-out. Through phone geolocation, we verify if the employee performs their check-in at assigned stores and completes their workday.

Workday Characteristics:

Fixed: 8 hours daily on weekdays.

Variable: Mobile routes where the same employee visits multiple stores in a day.

Problem: During a certain period, competitors began attracting our talent by offering slightly higher salaries. As a result, we faced a crisis where the service hours KPI started to decrease. This affected the full payment for the service and the performance bonus from the client.

Problem Analysis: The agency began searching for strategies to solve the issue, but as the analyst, I had the task of evaluating whether the KPI measurement was done correctly. I found that the client's goal wasn't calculated with great precision. They only calculated the weekly hours contracted from the agency and multiplied them by the weeks of the month to set their goal. However, they didn't account for factors like vacations, incapacities, natural disasters (such as flooding or tsunami season), vacant routes, and new personnel who only received their cellphones and began accounting for their hours one month after hiring. Many variables were not considered, which must be addressed for a more robust analysis of the goal and our achievement in this matter.

With this intention, and as we have all the records of vacations, incapacities, natural disasters, and the status of routes (whether vacant or occupied by new entry personnel), I decided to conduct a robust analysis. By including all this information, we can calculate the KPI goal in a much more accurate and effective way.

Query: Objetivos


/* the main source was pur asignation of stores, where are listed all the routes that we're responsible for, every route includes the stores that must be visited within this and the percentage of the weekly schedule assigned to every store it's described how much personal is assigned to every store. */

let
    Source = ASIGNACION,
/* Preparing the data for manipulation: 
   - Only keeping the important fields
   - Establishing the type of each field for further operations
   - Replacing values that may cause inconsistencies in data types with values that make sense and are consistent with the data type of the field */
    #"Removed Columns" = Table.RemoveColumns(Source,{"lunes", "martes", "miércoles", "jueves", "viernes", "sábado", "Estado", "Localidad", "SO", "Puesto", "Nivel", "Nombre", "Usuario ", "RETAIL X", "Lectura", "IR-RXL", "Correo", "Contraseña", "Usuario RETAIL X", "ID RETAIL X", "Contraseña RETAIL X", "OFICINA ENCARGADA", "LECTURA.1", "CAPACITADO IM-RXL", "Llave"}),
    #"Replaced Value" = Table.ReplaceValue(#"Removed Columns","Vacante","0",Replacer.ReplaceText,{"ID"}),
    #"Changed Type1" = Table.TransformColumnTypes(#"Replaced Value",{{"ID", Int64.Type}}),

/* Joining with our personnel table. This join will provide information on which routes are being attended by new personnel (those who did not yet have the tools for their hours to be accounted for in our system). */
    #"Merged Queries" = Table.NestedJoin(#"Changed Type1", {"ID"}, MAESTRO, {"ID SNAP"}, "BD", JoinKind.LeftOuter),
    #"Expanded BD" = Table.ExpandTableColumn(#"Merged Queries", "BD", {"FECHA INGRESO"}, {"BD.FECHA INGRESO"}),
/* Preparing the fields and the potentially different formats of the data to identify the new personnel */
    #"Added Custom1" = Table.AddColumn(#"Expanded BD", "CUSTOM.1", each if [ID]=0 then null else if [FECHA DE INGRESO]="-" then [BD.FECHA INGRESO] else [FECHA DE INGRESO]),
    #"Removed Columns2" = Table.RemoveColumns(#"Added Custom1",{"FECHA DE INGRESO", "BD.FECHA INGRESO"}),
    #"Renamed Columns" = Table.RenameColumns(#"Removed Columns2",{{"CUSTOM.1", "FECHA DE INGRESO"}}),
    #"Replaced Value1" = Table.ReplaceValue(#"Renamed Columns",null,#date(1900, 1, 1),Replacer.ReplaceValue,{"FECHA DE INGRESO"}),
    #"Changed Type" = Table.TransformColumnTypes(#"Replaced Value1",{{"FECHA DE INGRESO", type date}}),
/* identifyng the new personnel */
    #"Added Conditional Column" = Table.AddColumn(#"Changed Type", "NUEVO INGRESO", each if [FECHA DE INGRESO] >= #date(2024, 10, 12) then 1 else 0),
/* Identifying routes that were vacant */
    #"Added Custom" = Table.AddColumn(#"Added Conditional Column", "VACANTE", each if [ID] = 0 then 1 else 0),
/* Setting everything up to identify the time that must be deducted, as this time may vary depending on whether the routes were fixed or mobile */
    #"Valor reemplazado" = Table.ReplaceValue(#"Added Custom",#date(1900, 1, 1),null,Replacer.ReplaceValue,{"FECHA DE INGRESO"}),
/* Calculating the gross hours that a route must accomplish 
   (without adjustments due to incidents, vacancies, or being occupied by new personnel) */

    #"Personalizada agregada" = Table.AddColumn(#"Valor reemplazado", "OBJETIVO MES", each if ([Tipo Asignacion] = "Fijo" or [Tipo Asignacion] = "FIJO")  then 180*[Asignación HC] else 168.5*[Asignación HC]),
    #"Personalizada agregada1" = Table.AddColumn(#"Personalizada agregada", "A descontar VAC/NE", each if ([VACANTE]=1 or [NUEVO INGRESO]=1) then [OBJETIVO MES] else 0),
/* Left Join with our incidencies table (incapacities, vacations, natural disasters) 
   for further adjustments if a person had any of these during the month */

    #"Consultas combinadas" = Table.NestedJoin(#"Personalizada agregada1", {"ID"}, #"I,P,DN,VC RESUMEN", {"ID"}, "I,P,DN,VC RESUMEN", JoinKind.LeftOuter),
    #"Se expandió I,P,DN,VC RESUMEN" = Table.ExpandTableColumn(#"Consultas combinadas", "I,P,DN,VC RESUMEN", {"VC", "I", "P"}, {"VC", "I", "P"}),
    #"Valor reemplazado1" = Table.ReplaceValue(#"Se expandió I,P,DN,VC RESUMEN",null,0,Replacer.ReplaceValue,{"VC","I","P"}),
/* While joining with the incidencies table, we were getting repeated records, 
   as one person could have multiple days of vacation within the same month. 
   To avoid this and to clarify the total amount of hours to be adjusted, 
   the incidence columns were pivoted. */

    #"Otras columnas con anulación de dinamización" = Table.UnpivotOtherColumns(#"Valor reemplazado1", {"A descontar VAC/NE", "OBJETIVO MES", "VACANTE", "NUEVO INGRESO", "FECHA DE INGRESO", "Ejecutivo", "Gerente", "Estatus", "ID", "Asignación HC", "Ruta", "Tipo Asignacion", "Zona", "Region", "Nombre de la cuenta", "Formato", "Cadena", "Canal", "ID Tienda", "Agency"}, "Atributo", "Valor"),
    #"Personalizada agregada2" = Table.AddColumn(#"Otras columnas con anulación de dinamización", "Personalizado", each [Valor]*[Asignación HC]),
    #"Columnas quitadas" = Table.RemoveColumns(#"Personalizada agregada2",{"Valor"}),
    #"Columna dinamizada" = Table.Pivot(#"Columnas quitadas", List.Distinct(#"Columnas quitadas"[Atributo]), "Atributo", "Personalizado")
in
    #"Columna dinamizada"

![image](https://github.com/user-attachments/assets/781d8441-f54a-4106-883d-69a004c0e9e1)
