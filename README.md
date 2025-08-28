¡Perfecto, Enrique! Vamos a hacerlo solo con DAX y cumpliendo tus reglas:

Si desaparece de un corte al siguiente ⇒ ATENDIDO entre esos dos cortes (aunque no tenga registro “concluido”).

Si reaparece en un corte posterior ⇒ cuenta como NUEVO desde ese corte (no hereda el origen anterior).

Concluidos directos (solo en la hoja Concluidos con Fecha_estado y nunca figuraron Pendientes) se asignan al corte según su Fecha_estado.


Asumo:

Tabla única: Hechos_Snapshot

Columnas: ExpedienteID, Estado (PENDIENTE / CONCLUIDO), Corte (fecha del corte), Fecha_estado (en los concluidos)

Tabla de cortes: Dim_Corte con columna Corte (relacionada 1–* con Hechos_Snapshot[Corte])



---

1) Utilidades: “corte anterior” dinámico

Usaremos siempre “el corte anterior real” (tus cortes no son fijos).

Corte Anterior :=
VAR CorteActual = MAX ( Dim_Corte[Corte] )
RETURN
CALCULATE (
    MAX ( Dim_Corte[Corte] ),
    FILTER ( ALL ( Dim_Corte ), Dim_Corte[Corte] < CorteActual )
)

> No es una medida que muestres; solo te sirve como patrón dentro de otras medidas (copiamos la lógica CorteAnterior como variable).




---

2) Vista “MES A MES” (evolución entre dos cortes consecutivos)

2.1 Pendientes NUEVOS del corte (incluye reapariciones)

> Pendiente en el corte actual y NO estaba pendiente en el corte anterior.



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

2.2 Pendientes ARRASTRADOS en el corte

> Pendiente en el corte actual y SÍ estaba pendiente en el corte anterior.



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

2.3 ATENDIDOS entre cortes

> Pendiente en el corte anterior y ya no aparece como pendiente en el actual ⇒ atendido entre ambos.



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

2.4 Concluidos directos en el corte (nunca fueron pendientes)

> Expedientes que jamás aparecen como Pendientes y cuya primera Fecha_estado cae entre el corte anterior y el actual.



Concluidos DIRECTOS (por fecha estado) :=
VAR CorteActual   = MAX ( Dim_Corte[Corte] )
VAR CorteAnterior =
    CALCULATE ( MAX ( Dim_Corte[Corte] ),
        FILTER ( ALL ( Dim_Corte ), Dim_Corte[Corte] < CorteActual ) )
VAR CorteAnteriorAjustado = COALESCE ( CorteAnterior, DATE ( 1900, 1, 1 ) )
RETURN
CALCULATE (
    DISTINCTCOUNT ( Hechos_Snapshot[ExpedienteID] ),
    -- Nunca fue PENDIENTE en ningún corte
    CALCULATE (
        COUNTROWS ( Hechos_Snapshot ),
        ALLEXCEPT ( Hechos_Snapshot, Hechos_Snapshot[ExpedienteID] ),
        Hechos_Snapshot[Estado] = "PENDIENTE"
    ) = 0,
    -- Tiene fecha de estado y cae en (anterior, actual]
    NOT ISBLANK ( Hechos_Snapshot[Fecha_estado] ),
    MIN ( Hechos_Snapshot[Fecha_estado] ) >  CorteAnteriorAjustado,
    MIN ( Hechos_Snapshot[Fecha_estado] ) <= CorteActual
)

> Úsala en tarjetas o columnas apiladas junto a ATENDIDOS.




---

3) Vista “ACUMULADA POR COHORTES” (origen vs cortes sucesivos)

Definimos cohorte de origen como: expedientes pendientes en un corte y NO pendientes en el corte anterior (tu regla de “nuevo ciclo al reaparecer”).

La matriz será:

Filas: Dim_Corte[Corte] (corte origen de la cohorte)

Columnas: Dim_Corte[Corte] (corte de seguimiento)

Valores: medida de “pendientes de la cohorte en ese corte de seguimiento”


Pendientes de COHORTE (fila → columna) :=
VAR CorteFila =
    SELECTEDVALUE ( Dim_Corte[Corte] )               -- cohorte origen (filas)
VAR CorteCol  =
    MAX ( Dim_Corte[Corte] )                         -- corte seguimiento (columnas)
VAR CorteAnteriorFila =
    CALCULATE ( MAX ( Dim_Corte[Corte] ),
        FILTER ( ALL ( Dim_Corte ), Dim_Corte[Corte] < CorteFila ) )
RETURN
CALCULATE (
    DISTINCTCOUNT ( Hechos_Snapshot[ExpedienteID] ),
    -- (1) PERTENECE a la cohorte de CorteFila:
    --    estuvo PENDIENTE en CorteFila…
    CALCULATE (
        COUNTROWS ( Hechos_Snapshot ),
        Hechos_Snapshot[Estado] = "PENDIENTE",
        Hechos_Snapshot[Corte]  = CorteFila
    ) > 0
    &&
    --    …y NO estuvo PENDIENTE en el corte anterior a CorteFila
    CALCULATE (
        COUNTROWS ( Hechos_Snapshot ),
        Hechos_Snapshot[Estado] = "PENDIENTE",
        Hechos_Snapshot[Corte]  = CorteAnteriorFila
    ) = 0,
    -- (2) Y ESTÁ PENDIENTE en el corte de seguimiento (columna)
    Hechos_Snapshot[Estado] = "PENDIENTE",
    Hechos_Snapshot[Corte]  = CorteCol
)

> Esta medida llena la matriz de cohortes (filas = origen, columnas = seguimiento).
Verás cómo cada cohorte se va “apagando” con el tiempo conforme los casos se atienden.




---

4) Cómo usarlo en el informe

Página 1 – Evolución mes a mes

Segmentador: Dim_Corte[Corte] (selección única).

Tarjetas:

Pendientes NUEVOS (corte)

Pendientes ARRASTRADOS (corte)

ATENDIDOS (del anterior al actual)

Concluidos DIRECTOS (por fecha estado)


Tabla detalle filtrada por el corte seleccionado (ExpedienteID, Especialista, Procedimiento, etc.)

Gráfico columnas apiladas: Nuevos vs Arrastrados vs Atendidos.


Página 2 – Cohortes acumuladas

Matriz:

Filas = Dim_Corte[Corte] (origen)

Columnas = Dim_Corte[Corte] (seguimiento)

Valores = Pendientes de COHORTE (fila → columna)


(Opcional) Línea del tiempo con total pendientes por cohorte usando esta misma lógica.



---

Notas finas

Si el primer corte no tiene “anterior”, las fórmulas ya contemplan CorteAnteriorAjustado para no fallar.

Asegúrate de que Dim_Corte[Corte] esté relacionada con Hechos_Snapshot[Corte].

Normaliza Estado (“PENDIENTE” / “CONCLUIDO”) para evitar diferencias por mayúsculas.



---

Si quieres, te paso estas medidas en un bloque listo para copiar/pegar y te digo exactamente dónde crear cada una (todas en la tabla Hechos_Snapshot o en una tabla de medidas).

