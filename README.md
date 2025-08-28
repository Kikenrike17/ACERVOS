¡Hecho, Enrique! Te dejo clic por clic dónde crear cada cosa en Power BI Desktop y cómo usarla. No tienes que volver a buscar respuestas anteriores.


---

0) Preparación (una sola vez)

0.1 Crea la dimensión de cortes (Dim_Corte)

1. Cinta superior → Modelado → Nueva tabla.


2. Pega esto y Enter:



Dim_Corte = DISTINCT( Hechos_Snapshot[Corte] )

3. En el panel Campos, selecciona Dim_Corte[Corte] → en Propiedades pon Tipo de datos: Fecha.



0.2 Relaciona Dim_Corte con Hechos_Snapshot

1. Vista izquierda → icono Modelo.


2. Arrastra Dim_Corte[Corte] → sobre Hechos_Snapshot[Corte].


3. En el diálogo:

Cardinalidad: Uno a varios (1:)*

Dirección del filtro: Simple

Activar la relación → Aceptar.




0.3 (Opcional) Normaliza ESTADO

Si hay variantes de escritura:

1. Vista izquierda → icono Datos o en Modelado → Nueva columna dentro de Hechos_Snapshot:



EstadoNorm = UPPER( Hechos_Snapshot[Estado] )

2. Si la creas, usa "PENDIENTE"/"CONCLUIDO" en mayúsculas en todas las medidas.
(Si ya viene limpio, puedes seguir usando [Estado]).



0.4 Crea una tabla “Medidas” (para guardar todas las medidas juntas)

1. Cinta Inicio → Ingresar datos.


2. Nombre de la tabla: Medidas. Deja una columna vacía (o pon “Dummy” con 1 fila). → Cargar.


3. (Opcional) Borra la columna Dummy luego. Ahora todas las nuevas medidas las crearás seleccionando la tabla Medidas (quedarán allí ordenaditas).




---

1) Vista “Mes a mes” (Nuevos, Arrastrados, Atendidos, Concluidos directos)

> Importante: Antes de crear cada medida, haz clic en la tabla Medidas (en el panel Campos). Así las medidas quedarán agrupadas ahí.



1.1 Medidas DAX

Pendientes NUEVOS (corte)

1. Modelado → Nueva medida.


2. Pega:



Pendientes NUEVOS (corte) :=
VAR CorteActual   = MAX ( Dim_Corte[Corte] )
VAR CorteAnterior =
    CALCULATE ( MAX ( Dim_Corte[Corte] ),
        FILTER ( ALL ( Dim_Corte ), Dim_Corte[Corte] < CorteActual ) )
RETURN
CALCULATE (
    DISTINCTCOUNT ( Hechos_Snapshot[ExpedienteID] ),
    Hechos_Snapshot[Estado] = "PENDIENTE",
    Hechos_Snapshot[Corte]  = CorteActual,
    NOT (
        CALCULATE (
            COUNTROWS ( Hechos_Snapshot ),
            Hechos_Snapshot[Estado] = "PENDIENTE",
            Hechos_Snapshot[Corte]  = CorteAnterior
        ) > 0
    )
)

Pendientes ARRASTRADOS (corte)

1. Modelado → Nueva medida.


2. Pega:



Pendientes ARRASTRADOS (corte) :=
VAR CorteActual   = MAX ( Dim_Corte[Corte] )
VAR CorteAnterior =
    CALCULATE ( MAX ( Dim_Corte[Corte] ),
        FILTER ( ALL ( Dim_Corte ), Dim_Corte[Corte] < CorteActual ) )
RETURN
CALCULATE (
    DISTINCTCOUNT ( Hechos_Snapshot[ExpedienteID] ),
    Hechos_Snapshot[Estado] = "PENDIENTE",
    Hechos_Snapshot[Corte]  = CorteActual,
    CALCULATE (
        COUNTROWS ( Hechos_Snapshot ),
        Hechos_Snapshot[Estado] = "PENDIENTE",
        Hechos_Snapshot[Corte]  = CorteAnterior
    ) > 0
)

ATENDIDOS (del anterior al actual)

1. Modelado → Nueva medida.


2. Pega:



ATENDIDOS (del anterior al actual) :=
VAR CorteActual   = MAX ( Dim_Corte[Corte] )
VAR CorteAnterior =
    CALCULATE ( MAX ( Dim_Corte[Corte] ),
        FILTER ( ALL ( Dim_Corte ), Dim_Corte[Corte] < CorteActual ) )
RETURN
CALCULATE (
    DISTINCTCOUNT ( Hechos_Snapshot[ExpedienteID] ),
    -- estaba PENDIENTE en el anterior
    CALCULATE (
        COUNTROWS ( Hechos_Snapshot ),
        Hechos_Snapshot[Estado] = "PENDIENTE",
        Hechos_Snapshot[Corte]  = CorteAnterior
    ) > 0,
    -- y NO está PENDIENTE en el actual
    NOT (
        CALCULATE (
            COUNTROWS ( Hechos_Snapshot ),
            Hechos_Snapshot[Estado] = "PENDIENTE",
            Hechos_Snapshot[Corte]  = CorteActual
        ) > 0
    )
)

Concluidos DIRECTOS (por fecha estado)

1. Modelado → Nueva medida.


2. Pega:



Concluidos DIRECTOS (por fecha estado) :=
VAR CorteActual   = MAX ( Dim_Corte[Corte] )
VAR CorteAnterior =
    CALCULATE ( MAX ( Dim_Corte[Corte] ),
        FILTER ( ALL ( Dim_Corte ), Dim_Corte[Corte] < CorteActual ) )
VAR CorteAnteriorAdj = COALESCE ( CorteAnterior, DATE ( 1900, 1, 1 ) )

VAR ConcluByExp =
    SUMMARIZE(
        FILTER(
            ALL(Hechos_Snapshot),
            Hechos_Snapshot[Estado] = "CONCLUIDO"
                && NOT ISBLANK(Hechos_Snapshot[Fecha_estado])
        ),
        Hechos_Snapshot[ExpedienteID],
        "FirstFechaConclu", MIN(Hechos_Snapshot[Fecha_estado])
    )

VAR PendingIDs =
    SUMMARIZE(
        FILTER(ALL(Hechos_Snapshot), Hechos_Snapshot[Estado] = "PENDIENTE"),
        Hechos_Snapshot[ExpedienteID]
    )

VAR DirectosEnCorte =
    FILTER(
        ConcluByExp,
        [FirstFechaConclu] >  CorteAnteriorAdj
            AND [FirstFechaConclu] <= CorteActual
            AND NOT CONTAINS(
                PendingIDs,
                Hechos_Snapshot[ExpedienteID], Hechos_Snapshot[ExpedienteID]
            )
    )
RETURN
COUNTROWS( DirectosEnCorte )

1.2 Arma la página “Mes a Mes”

1. Inserta un Segmentador → arrastra Dim_Corte[Corte].

Formato del segmentador → Selección única: Activado.

Ordena por fecha (descendente si quieres ver el último primero).



2. Inserta 4 Tarjetas (visual Card) y arrastra estas medidas:

Pendientes NUEVOS (corte)

Pendientes ARRASTRADOS (corte)

ATENDIDOS (del anterior al actual)

Concluidos DIRECTOS (por fecha estado)



3. Inserta una Tabla de detalle: agrega columnas de Hechos_Snapshot (ExpedienteID, Especialistas, Procedimiento, etc.). El segmentador filtra todo.




---

2) Vista “Cohortes” (origen vs seguimiento)

2.1 Crea la tabla de columnas desconectada (Dim_Corte_Col)

1. Modelado → Nueva tabla:



Dim_Corte_Col = DISTINCT( Hechos_Snapshot[Corte] )

2. No crees relación de Dim_Corte_Col con nadie (debe estar desconectada).



2.2 Crea la medida de cohorte

1. Selecciona la tabla Medidas → Nueva medida.


2. Pega:



Pendientes de COHORTE (fila → columna) :=
VAR CorteFila = SELECTEDVALUE( Dim_Corte[Corte] )            -- origen (filas)
VAR CorteAnteriorFila =
    CALCULATE( MAX(Dim_Corte[Corte]),
        FILTER( ALL(Dim_Corte), Dim_Corte[Corte] < CorteFila ) )
VAR CorteCol = SELECTEDVALUE( Dim_Corte_Col[Corte] )          -- seguimiento (columnas)
RETURN
CALCULATE(
    DISTINCTCOUNT( Hechos_Snapshot[ExpedienteID] ),

    -- (1) PERTENECE a la cohorte del CorteFila:
    CALCULATE(
        COUNTROWS( Hechos_Snapshot ),
        Hechos_Snapshot[Estado] = "PENDIENTE",
        Hechos_Snapshot[Corte]  = CorteFila
    ) > 0
    &&
    CALCULATE(
        COUNTROWS( Hechos_Snapshot ),
        Hechos_Snapshot[Estado] = "PENDIENTE",
        Hechos_Snapshot[Corte]  = CorteAnteriorFila
    ) = 0,

    -- (2) Está PENDIENTE en el corte de seguimiento (columna):
    Hechos_Snapshot[Estado] = "PENDIENTE",
    TREATAS( VALUES(Dim_Corte_Col[Corte]), Hechos_Snapshot[Corte] )
)

2.3 Arma la página “Cohortes”

1. Inserta un visual Matriz.


2. Filas → arrastra Dim_Corte[Corte] (origen).


3. Columnas → arrastra Dim_Corte_Col[Corte] (seguimiento).


4. Valores → arrastra Pendientes de COHORTE (fila → columna).


5. (Opcional) Formato condicional por color para ver cómo “se apaga” cada cohorte.




---

3) Consejos rápidos

Si tu Estado viene como “Pendiente/Concluido” en distintos casos, usa EstadoNorm y reemplaza en las medidas "PENDIENTE" por "PENDIENTE" sobre esa nueva columna.

Si un corte es el primero, las medidas ya contemplan que no hay “anterior”.

Si no te aparece el resultado esperado, verifica:

La relación Dim_Corte → Hechos_Snapshot.

Que el segmentador use Dim_Corte[Corte] (no el campo de Hechos).

Que Corte esté en formato Fecha.




---

¿Quieres que te haga un mini checklist visual (capturas simuladas) de dónde está Nueva tabla, Nueva medida, Segmentador, Matriz y Tarjeta, para que lo ubiques en pantalla?

