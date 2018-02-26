# Readme SAS CdM

El SAS está compuesto por 3 flujos principales:
1.	**Variables y Macros**
2.	[**Modelo Capacidad de Pago**](#1-modelo-de-capacidad-de-pago-)
3.	[**Modelo de Contactabilidad**](#2-modelo-de-contactabilidad-)

# 1. Modelo de capacidad de pago [↩](#readme-sas-cdm)

El modelo de capacidad de pago está compuesto por 3 programas principales:
1. [**Vars y macros**](#11-vars-y-macros)
2. [**Transformer**](#12-transformer)
3. [**Modeler**](#13-modeler)

## 1.1. Vars y macros [↩](#1-modelo-de-capacidad-de-pago)

* Parte de la tabla de entrada del modelo, añade el número de fila<sup name="a1-1-1">[[1]](#f1-1-1)</sup> y calcula:
  `CAPACIDAD2 = SIGN(CAPACIDAD)*LOG(ABS(CAPACIDAD)+1)`<sup id="a1-1-2">[[2]](#f1-1-2)</sup>  

* Sobre la tabla de output del paso anterior, renombra `CAPACIDAD2` &rightarrow; `CAPACIDAD`

* Cuenta el número de registros agrupando por `&ID_FECHA`<sup id="a1-1-3">[[3]](#f1-1-3)</sup>, `&IDENTIFICADOR`<sup id="a1-1-4">[[4]](#f1-1-4)</sup>

* Define varias macros que encapsulan distintos modelos de clasificación:
    - [**Lasso regression**](#lasso-regression-5)
    - [**Treeboost regression**](#treeboost-regression)
    - [**Tree regression**](#tree-regression)
    - [**PCA Regression**](#pca-regression)
    - [**Cluster regression**](#cluster-regression)

### Lasso regression<sup id="a1-1-5">[[5]](#f1-1-5)</sup> [↩](#11-vars-y-macros):

`LASSO_REGRESSION_MODEL(DATA, FLAG, MODELO)`

Incluye a su vez los siguientes componentes (macros):
- Training set:

 `LASSO_TRAIN(&DATA, &FLAG, &WEIGHT, &MODELO)`

- Test set<sup id="a1-1-6">[[6]](#f1-1-6)</sup>:

 `LASSO_PREDICT(&DATA, &FLAG, &MODELO, PRED_TEST_LASSO)`

### Treeboost regression [↩](#11-vars-y-macros):

`TREEBOOST_REGRESSION_MODEL(&TRAIN_DATA, &TEST_DATA, &FLAG, &WEIGHT, &MODELO, &RESULTADOS_TREEBOOST, &DEPTH)`  

* Hace un bucle de 7 iteraciones para buscar el número de iteraciones del árbol que maximice el IP del test. Las iteraciones se calcularán como:

 `&ITERACIONES = 2^I`<sup id="a1-1-7">[[7]](#f1-1-7)</sup>

* En cada una de las iteraciones, ejecutará los siguientes componentes (macros):
    - Training set:

    `REGTREEBOOST_TRAIN(&TRAIN_DATA, &FLAG, &WEIGHT, &MODELO_GBM, &ITERATIONS, &DEPTH)`

    - Test set:

    `REGTREEBOOST_PREDICT(&TEST_DATA, &FLAG, &MODELO_GBM, PRED_TEST_TREEBOOST)`

* Tras ejecutar cada iteración, se guardan las siguientes métricas en `&RESULTADOS_TREEBOOST`:
    - **Franja:** `&AUX_FRANJAS`
    - **Iteraciones:** `&ITERACIONES`
    - **R<sup>2</sup> train:** `&R2_TREEBOOST_TRAIN`
    - **R<sup>2</sup> test:** `&R2_TREEBOOST_TEST`
    - **RMSE<sup id="a1-1-8">[[8]](#f1-1-8)</sup> train:** `&RMSE_TREEBOOST_TRAIN`
    - **RMSE test:** `&RMSE_TREEBOOST_TEST`

 … y se selecciona el hiperparámetro `&ITERACIONES` que minimice el RMSE test<sup id="a1-1-9">[[9]](#f1-1-9)</sup>.

* Una vez se tienen las iteraciones óptimas, se entrena de nuevo el modelo, cambiando algunos parámetros:

 `REGTREEBOOST_TRAIN(&FULL_DATA, &FLAG, &WEIGHT, &MODELO_GBM, &BEST_ITER, 10)`<sup id="a1-1-10">[[10]](#f1-1-10)</sup>

### Tree regression [↩](#11-vars-y-macros):

`TREE_REGRESSION_MODEL(TRAIN_DATA, TEST_DATA, FLAG, MODELO, RESULTADOS_TREE)`

* Hace un bucle de 10 iteraciones para encontrar la profundidad óptima del árbol:
`&DEPTH = 5*I`<sup id="a1-1-11">[[11]](#f1-1-11)</sup>

* En cada una de las iteraciones, ejecutará los siguientes componentes (macros):
    - **Training set:**
    `REGTREE_TRAIN(&TRAIN_DATA, &FLAG, &WEIGHT, &MODELO_ARBOL, &DEPTH)`

    - **Test set:**
    `REGTREE_PREDICT(&TEST_DATA, &FLAG,  &WEIGHT, &MODELO_ARBOL, PRED_TEST_TREE)`

* Tras ejecutar cada iteración, se guardan las siguientes métricas en `&RESULTADOS_TREE`:
    - **Franja:** `&AUX_FRANJAS`
    - **Depth:** `&DEPTH`
    - **R<sup>2</sup> train:** `&R2_TREE_TRAIN`
    - **R<sup>2</sup> test:** `&R2_TREE_TEST`
    - **RMSE train:** `&RMSE_TREE_TRAIN`
    - **RMSE test:** `&RMSE_TREE_TEST`

 … y se selecciona el hiperparámetro `&DEPTH` que minimice el RMSE test.

* Una vez se tienen las iteraciones óptimas, se entrena de nuevo el modelo, cambiando algunos parámetros:
`REGTREE_TRAIN(&FULL_DATA, &FLAG, &WEIGHT, &MODELO_ARBOL, &BEST_DEPTH)`<sup id="a1-1-12">[[12]](#f1-1-12)</sup>

### PCA regression [↩](#11-vars-y-macros):

`PCA_REGRESSION_MODEL(TRAIN_DATA, TEST_DATA, FLAG, TRANSF_PCA, MODELO_PCA, RESULTADOS_PCA)`

* Se calcula el número máximo de componentes: `&MAX_NUM_COMPONENTS`.

* Hace un bucle de 20 iteraciones para calcular el número de componentes principales (PC) utilizados en el modelo:
`&NUM_PC = &PC*&MAX_NUM_COMPONENTS/20`<sup id="a1-1-13">[[13]](#f1-1-13)</sup>

* En cada una de las iteraciones, ejecutará los siguientes componentes (macros):

    - **Training set:**
    `REGRESSION_PCA_TRAIN(TABLE_IN=&TRAIN_DATA, FLAG=&FLAG, N=&NUM_PC, MODEL=&MODELO_PCA, TRANS_PCA=&TRANSF_PCA)`

    - **Test set:**
    `REGRESSION_PCA_PREDICT(&TEST_DATA, &FLAG, &TRANSF_PCA, &MODELO_PCA, PRED_TEST_PCA)`

* Tras ejecutar cada iteración, se guardan las siguientes métricas en &RESULTADOS_PCA:
    - **Franja:** `&AUX_FRANJAS`
    - **Components:** `&NUM_PC`
    - **R<sup>2</sup> train:** `&R2_PCA_TRAIN`
    - **R<sup>2</sup> test:** `&R2_PCA_TEST`
    - **RMSE train:** `&RMSE_PCA_TRAIN`
    - **RMSE test:** `&RMSE_PCA_TEST`

 … y se selecciona el hiperparámetro `&NUM_PC` que minimice el RMSE test.

* Una vez se tiene el óptimo de componentes principales, se entrena de nuevo el modelo, cambiando algunos parámetros:
`REGRESSION_PCA_TRAIN(&FULL_DATA, &FLAG, &BEST_PC, &MODELO_PCA, &TRANSF_PCA)`<sup id="a1-1-14">[[14]](#f1-1-14)</sup>

### Cluster regression [↩](#11-vars-y-macros):

`CLUSTER_REGRESSION_MODEL(TRAIN_DATA, TEST_DATA, FLAG, WEIGHT, MODELO_MOD1, SEGMENTO, AUX_FRANJAS, TABLA_VARCLUSTER, PRED_VAR, BETAS_MOD1, TABLE_PRED_MOD1, RESULTADOS_PCA)`

* Hace un bucle de 14 iteraciones para calcular el IP univariante mínimo necesario para seleccionar una variable:
`&IND_MIN = 0.07 - &I * 0.005`

* En cada una de las iteraciones, ejecutará los siguientes componentes (macros):
    - **Selección variables:**
    `SELECCION_VARS(“REG”, &FLAG, &WEIGHT, &TRAIN_DATA, &TABLA_VARCLUSTER, &IND_MIN, TABLA_MODULO1, &AUX_FRANJAS)`

    - **Training set:**
    `REGRESSION_TRAIN(TABLA_MODULO1, &FLAG, &WEIGHT, &AUX_FRANJAS, &SEGMENTO, &MODELO_MOD1, &BETAS_MOD1, &TABLE_PRED_MOD1)`

    - **Test set:**
    `REGRESSION_PREDICT(&TEST_DATA, &FLAG, &MODELO_MOD1, PRED_TEST_CLUSTER, &PRED_VAR)`

* Tras ejecutar cada iteración, se guardan las siguientes métricas en `&RESULTADOS_CLUSTER`:
    - **Franja:** `&AUX_FRANJAS`
    - **Módulo:** `“UNICO”`
    - **R<sup>2</sup> min:**<sup id="a1-1-15">[[15]](#f1-1-15)</sup> `&IND_MIN`
    - **R<sup>2</sup> train:** `&R2_TRAIN`
    - **R<sup>2</sup> test:** `&R2_TEST`
    - **RMSE train:** `&RMSE_TRAIN`
    - **RMSE test:** `&RMSE_TEST`

 … y se selecciona el hiperparámetro `&IND_MIN` que minimice el RMSE test.

* Una vez se tiene el IP univariante óptimo, se entrena de nuevo el modelo, cambiando algunos parámetros:
    - `SELECCION_VARS(“REG”, &FLAG, &WEIGHT, &FULL_DATA, &TABLA_VARCLUSTER, &BEST_R2_MIN, TABLA_MODULO1, &AUX_FRANJAS)`

    - `REGRESSION_TRAIN(TABLA_MODULO1, &FLAG, &WEIGHT, &AUX_FRANJAS, &SEGMENTO, &MODELO_MOD1, &BETAS_MOD1, &TABLE_PRED_MOD1)`<sup id="a1-1-16">[[16]](#f1-1-16)</sup>

## 1.2. Transformer [↩](#1-modelo-de-capacidad-de-pago)

* Define la macro `TRANSFORMER` que da nombre al programa:
`TRANSFORMER(BASE_A_PARTIR, FLAG, WEIGHT, SEGMENT, TRAIN_TEST_SPLIT)`

    - Ejecuta las macros `INICIALIZACION_R2(&TABLA_R2)` e `INICIALIZACION_CLUSTER(&TABLA_VARCLUSTER)`

    - Hace un bucle de 1 a `&FRANJAS_LEN`<sup id="a1-2-1">[[1]](#f1-2-1)</sup>. En cada iteración, ejecuta los siguientes componentes (macros):

        - Particionado por franja temporal:
        `INICIALIZAR_TABLAS(&AUX_FRANJAS, &SEGMENTO, CAP)`

        - Filtrado por días de impago para reducir imbalancement:
        `FILTRAR_DATOS(&BASE_A_PARTIR, &TABLA_FILTRO_DIAS, &AUX_FRANJAS_CORTE, &VAR_NOT_ENTERING, &SEGMENT, &AUX_FRANJAS)`

        - Genera la tabla `&FULL_DATA` a partir de la salida del paso anterior (`&TABLA_FILTRO_DIAS`), y revierte la aplicación del logaritmo sobre la capacidad para poder calcular la mediana sobre el resultado:
        `&MEDIA = MEDIAN(SIGN(&FLAG)*EXP(ABS(&FLAG))-1)`<sup id="a1-2-2">[[2]](#f1-2-2)</sup>

        - Filtra casos anómalos de la base, conservando lo que se queda entre los percentiles 5 y 90:
        `REMOVE_OUTLIERS(&FULL_DATA, CAPACIDAD, P_INF=P5, P_SUP=P90)`

        - Calcula el R2 univariante para las variables de la muestra:
        `CALCULO_R2_UNIVARIANTE(&FULL_DATA, &FLAG, &AUX_FRANJAS_CORTE)`

        - Calcula clusters de variables correlacionadas del dataset: `CLUSTERS_Y_R2(&FULL_DATA, &AUX_FRANJAS_CORTE, &FLAG)`

        - Cálculo de PCA para las variables del dataset:
        `PCA_TRAIN(FULL_DATA, &TRANSF_PCA, PCData, &FLAG)`

        - División de la tabla en train set y test set:
        `TRAIN_TEST_SPLIT(&FULL_DATA, &TRAIN_DATA, &TEST_DATA, &TRAIN_TEST_SPLIT, &FLAG)`

* Llama la macro `TRANSFORMER` que ha definido en el paso anterior:
`TRANSFORMER(&TABLA_MODELO, CAPACIDAD, WEIGHTS, &SEGMENTO, 0.8)`

## 1.3. Modeler [↩](#1-modelo-de-capacidad-de-pago)

* Define la macro `MODELER` que da nombre al programa:
`MODELER(BASE_A_PARTIR, FLAG, WEIGHT)`

    - Ejecuta las macros de inicialización de diversos modelos (PCA, cluster, tree, treeboost):
        - `INI_RESULTADOS_PCA(&RESULTADOS_PCA)`
        - `INI_RESULTADOS_CL(&RESULTADOS_CLUSTER)`
        - `INI_RESULTADOS_TREE(&RESULTADOS_TREE)`
        - `INI_RESULTADOS_TREEBOOST(&RESULTADOS_TREEBOOST)`

    - Hace un bucle iterando de 1 a `&FRANJAS_LEN` para particionar por franja temporal, y realiza los siguientes pasos:

        - Calcula `&AUX_FRANJAS = &I`, y fija el valor de `&AUX_FRANJAS_CORTE = TOT`

        - Lanza los 5 modelos de regresión descritos en el punto 1.1:

            - `PCA_REGRESSION_MODEL(&TRAIN_DATA, &TEST_DATA, &FLAG, &TRANSF_PCA, &MODELO_PCA, &RESULTADOS_PCA)`

            - `LASSO_REGRESSION_MODEL(&FULL_DATA, &FLAG, &MODELO_LASSO)`

            - `CLUSTER_REGRESSION_MODEL(&TRAIN_DATA, &TEST_DATA, &FLAG, &WEIGHT, &MODELO_MOD1, &SEGMENTO, &AUX_FRANJAS, &TABLA_VARCLUSTER, &PRED_VAR, &BETAS_MOD1, &TABLE_PRED_MOD1, &RESULTADOS_PCA)`

            - `TREE_REGRESSION_MODEL(&TRAIN_DATA, &TEST_DATA, &FLAG, &MODELO_ARBOL, &RESULTADOS_TREE)`

            - `TREEBOOST_REGRESSION_MODEL(&TRAIN_DATA, &TEST_DATA, &FLAG, &WEIGHT, &MODELO_GBM, &RESULTADOS_TREEBOOST, 10)`

* Ejecuta la macro `MODELER` definida en el paso anterior:
`MODELER(&TABLA_MODELO, CAPACIDAD, WEIGHT_AUX)`

# 2. Modelo de contactabilidad [↩](#readme-sas-cdm)

## 2.1. Variables:

* Define una serie de parámetros del proceso del módulo 2

* Define la macro `CHECK_VARS(TAULA, FLAG, NOT_IN_MODEL)`:
    - Filtra la tabla de entrada haciendo `DROP` de las columnas `&NOT_IN_MODEL`.

    - Genera 4 tablas en total:

        - **Variables categóricas:**

            1. Crea la tabla `AUX_MODEL_CAT`, manteniendo las columnas de la tabla de entrada que correspondan a variables categóricas. Guarda los nombres de dichas columnas en `&VARS_MODEL_CAT`.

            2. Crea la tabla `AUX_MODEL_CAT_NO_FLAG`, idéntica a la anterior pero excluyendo la columna `&FLAG` (el target del modelo). Guarda los nombres de dichas columnas en `&VARS_MODEL_CAT_NO_FLAG`.

        - **Variables numéricas:**

            1.	Crea la tabla `AUX_MODEL_NUM`, manteniendo las columnas de la tabla de entrada que correspondan a variables numéricas. Guarda los nombres de dichas columnas en `&VARS_MODEL_NUM`.

            2.	Crea la tabla `AUX_MODEL_NUM_NO_FLAG`, idéntica a la anterior pero excluyendo la columna `&FLAG` (el target del modelo). Guarda los nombres de dichas columnas en `&VARS_MODEL_NUM_NO_FLAG`.

    - Ejecuta `PROC DMDB` con la siguiente estructura:
    ```
    PROC DMDB DATA=&TAULA
    	VAROUT=WORK.NUMERICAL_VARS
    	CLASSOUT=WORK.CLASSIFICATION_VARS;
    	CLASS &VARS_MODEL_CAT;
    	TARGET &FLAG;
    	VAR &VARS_MODEL_NUM;
    RUN;
    ```
    A continuación se describe cada paso del código anterior:

     `PROC DMDB DATA=&TAULA`:
     > El objetivo de `PROC DMDB` es generar un catálogo de metadatos sobre los datos de entrada, basándose en los roles que juegan las variables en el modelo.

     `VAROUT=WORK.NUMERICAL_VARS`
     > Indica la tabla en la que se guardarán los metadatos de las variables numéricas.

     `CLASSOUT=WORK.CLASSIFICATION_VARS`
     > Análoga a la anterior, indica la tabla en la que se guardarán los metadatos de las variables de clasificación.

     `CLASS &VARS_MODEL_CAT`
     > Variables cuyas combinaciones definen subgrupos (“buckets”) de análisis. Para cada una de ellas, se genera la siguiente información: categoría (class level value), frecuencia e información de ordenación.

     `TARGET &FLAG`
     > Identifica variable target del modelo, que deberá incluirse también en las listas de variables VAR o CLASS.

     `VAR &VARS_MODEL_NUM`
     > Identifica variables de análisis y su orden en los resultados. El catálogo de metadatos contiene los siguientes estadísticos: `N`, `NMISS`, `MIN`, `MAX`, `SUM`, `SUMWGT`, `CSS`, `USS`, `STD`, `SKEWNESS`, y `KURTOSIS`.

    - Inserta en la variable `&CONSTANT_VARS` los nombres de las variables numéricas de la tabla `WORK.NUMERICAL_VARS` generada en el paso anterior que cumplan la condición `MIN=MAX`.

    - Crea la tabla tratada del modelo (`&TABLA_MODELO`) a partir de la tabla de entrada del proceso (`&TABLA_ENTRADA`), haciendo `DROP` de las variables constantes `&CONSTANT_VARS`<sup id="a2-1-1">[[1]](#f2-1-1)</sup>.

    - Ejecuta la macro `CHECK_CATEGORICS(&TABLA_MODELO)` sobre la tabla generada en el paso anterior.

## 2.2. Transformer:

* Define la macro `TRANSFORMER` que da nombre al programa:
`TRANSFORMER(BASE_A_PARTIR, FLAG, WEIGHT, MIN_DIAS, MAX_DIAS, SEGMENT, TRAIN_TEST_SPLIT)`:
    - Hace un bucle de 1 a `&FRANJAS_LEN`  para generar una partición por franja temporal. En cada iteración, ejecuta los siguientes componentes (macros):
        - Genera las variables `&AUX_FRANJAS` y `&AUX_FRANJAS_CORTE`.

        - Genera otras variables con nombres de tablas de salida del proceso, segmentadas por franja temporal:

            - `&TABLA_PARAMETRIZ = CON_TABLA_WOE_&AUX_FRANJAS._&SEGMENTO`

            - `&TABLA_IP = CON_TABLA_IP_&AUX_FRANJAS._&SEGMENTO`

            - `&TABLA_VARCLUSTER = CON_TABLA_VARCLUSTER_&AUX_FRANJAS._&SEGMENTO`

        - Ejecuta las siguientes macros con las tablas de entrada calculadas en el paso anterior:

            - `INICIALIZACION_WOE(&TABLA_PARAMETRIZ)`

            - `INICALIZACION_IP(&TABLA_IP)`

            - `INICIALIZACION_CLUSTERS(&TABLA_VARCLUSTER)`

        - Inicializa las tablas del proceso llamando a la macro
        `INICIALIZAR_TABLAS(&AUX_FRANJAS, &SEGMENTO, CON)`

        - Se filtra la tabla por días de impago:
        `FILTRAR_DATOS_DIAS(&BASE_A_PARTIR, &TABLA_FILTRO_DIAS, &MIN_DIAS, &MAX_DIAS, &AUX_FRANJAS_CORTE, &VAR_NOT_ENTERING, &SEGMENT)`

        - Se reduce el imbalancement:
        `IMBALANCE_REDUCTION(&TABLA_FILTRO_DIAS, &FULL_DATA, 0.1, &FLAG)`

        - Hay una serie de transformaciones comentadas porque son necesarias únicamente para modelos lineales. Dado que no se están usando modelos lineales, se excluyen de la macro `TRANSFORMER`:

            - Cálculo del WOE (Weight Of Evidence) para las variables categóricas:
            `TRANSF_CAT_WOE(WORK, &FULL_DATA, &AUX_FRANJAS_CORTE, &FLAG)`

            - Cálculo del IP univariante para las variables:
            `CALCULO_IP_UNIVARIANTE(&FULL_DATA, &FLAG, &AUX_FRANJAS_CORTE)`

            - Cálculo de clusters de variables del datatrain (variables correlacionadas):
            `CLUSTERS_Y_IP(&FULL_DATA., &AUX_FRANJAS_CORTE., &FLAG.)`

            - Cálculo del PCA para las variables
            `PCA_TRAIN(&FULL_DATA., &TRANSF_PCA., PCData, &FLAG.)`

        - Se estratifican los datos de train y test, y se divide la tabla de entrada en train set y test set:
        `STRAT_TRAIN_TEST_SPLIT(&FULL_DATA., &TRAIN_DATA., &TEST_DATA., 0.8, &FLAG.)`

        - Hace `DROP` de la tabla `&TABLA_FILTRO_DIAS`, que se usaba para la reducción de imbalancement en un paso anterior.

* Llama a la macro `TRANSFORMER` que ha definido en el paso anterior:
`TRANSFORMER(&TABLA_MODELO., FLAGRPC, WEIGHTS, 0, 90, &SEGMENTO_TABLA., 0.8)`

## 2.3. Modeler:

* Define las macros correspondientes a varios modelos de clasificación:
    - [**Treebost classification**](#treeboost-classification-model)
    - [**Tree classification**](#tree-classification-model)
    - [**PCA classification**](#pca-classification-model)
    - [**Cluster classification**](#cluster-classification-model)

### Treeboost classification model [↩](#23-modeler)

### Tree classification model [↩](#23-modeler)

### PCA classification model [↩](#23-modeler)

### Cluster classification model [↩](#23-modeler)

---
**Notas / comentarios sección 1.1. Vars y macros**

<a name="f1-1-1"><sup>[1]</a></sup> Si sólo se usa para contar y calcular `WEIGHT_AUX` no haría falta incorporar el campo; `COUNT(1)` ya serviría. [↩](#a1-1-1)

<a id="f1-1-2"><sup>[2]</a></sup> ¿En qué unidades está denominada la capacidad?
Se le suma 1 por si la capacidad es 0, para que el logaritmo no dé error.
Entiendo que el `LOG()` es para limitar el peso de los valores muy elevados en el modelo. [↩](#a1-1-2)

<a id="f1-1-3"><sup>[3]</a></sup> `ID_FCH_DATOS` [↩](#a1-1-3)

<a id="f1-1-4"><sup>[4]</a></sup> `ID_EXPEDIENTE_GCR` [↩](#a1-1-4)

<a id="f1-1-5"><sup>[5]</a></sup> i.e. Regularized Linear Regression [↩](#a1-1-5)

<a id="f1-1-6"><sup>[6]</a></sup> Dice que se busca la profundidad óptima del árbol. Entiendo que es un hiperparámetro; ¿se está calibrando directamente sobre test set o se están haciendo cross-validation set y test set por separado? [↩](#a1-1-6)

<a id="f1-1-7"><sup>[7]</a></sup> ¿Por qué la función que calcula la profundidad es la exponencial con base 2? [↩](#a1-1-7)

<a id="f1-1-8"><sup>[8]</a></sup> Root-Mean-Square Error (Deviation) [↩](#a1-1-8)

<a id="f1-1-9"><sup>[9]</a></sup> ¿Se está haciendo cross-validation? [↩](#a1-1-9)

<a id="f1-1-10"><sup>[10]</a></sup> Usando el dataset entero, las iteraciones óptimas según el hiperparámetro seleccionado tras minimizar RMSE en el Validation set [↩](#a1-1-10)

<a id="f1-1-11"><sup>[11]</a></sup> ¿Por qué `5*i`? ¿Por qué en treeboost es exponencial base 2 y ahora en el tree regression model se multiplica por 5? [↩](#a1-1-11)

<a id="f1-1-12"><sup>[12]</a></sup> Usando el dataset entero, la profundidad óptima según el hiperparámetro seleccionado tras minimizar RMSE en el Validation set [↩](#a1-1-12)

<a id="f1-1-13"><sup>[13]</a></sup> ¿Por qué esta fórmula? ¿Por qué se divide entre 20? [↩](#a1-1-13)

<a id="f1-1-14"><sup>[14]</a></sup> Se modifica la tabla de entrada para capturar el dataset entero y se lanza la regresión con el número óptimo de componentes principales [↩](#a1-1-14)

<a id="f1-1-15"><sup>[15]</a></sup> Entiendo que se deja el nombre R<sup>2</sup> por conveniencia, ya que `IND_MIN` hace referencia al IP univariante mínimo necesario para que una variable pueda entrar en un cluster del modelo. [↩](#a1-1-15)

<a id="f1-1-16"><sup>[16]</a></sup> Se modifica la tabla de entrada para capturar el dataset entero y se lanza la regresión con el número óptimo de componentes principales [↩](#a1-1-16)

---
**Notas / comentarios sección 1.2. Transformer**

<a id="f1-2-1"><sup>[1]</a></sup> De momento, `&FRANJAS_LEN = 1` [↩](#a1-2-1)

<a id="f1-2-2"><sup>[2]</a></sup> `&FLAG = CAPACIDAD` [↩](#a1-2-2)

---
**Notas / comentarios sección 2.1. Variables**

<a id="f2-1-1"><sup>[1]</a></sup> En la V6 este paso está comentado, por lo que no se están eliminando dichas variables de la tabla del modelo. [↩](#a2-1-1)

---
**Notas / comentarios sección 2.2. Transformer**

<a id="f2-2-1"><sup>[1]</a></sup> De momento, `&AUX_FRANJAS` y `&AUX_FRANJAS_CORTE = TOT (V6)` [↩](#a2-2-1)

---

Bla bla <sup id="ta1">[[1]](#tf1)</sup>
<a id="tf1"><sup>[1]</a></sup> Footnote content here. [↩](#ta1)
