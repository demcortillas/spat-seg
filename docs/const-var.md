>Construcción de las variables de variables a partir de la base de datos del censo

Importar paquetes a ocupar

```python
import pandas as pd
import geopandas as gpd
from functool import reduce
```

Muchas de las veces la capacidad ram de los computadores no es capaz de cargar la base de microdatos del censo, por lo que es recomendable pasar los .csv que entrega INE a formato .db y luego exportar lo que necesitamos de ahí a un .csv

```python
BD='BD.db'
COMUNAS_PERSONAS=sql2df(BD,"SELECT * FROM PERSONAS WHERE COMUNA=8101 OR COMUNA=8102 OR COMUNA=8103 OR COMUNA=8108 OR COMUNA= 8110 OR COMUNA=8112")
COMUNAS_PERSONAS.to_csv('BD_P_AMC.csv',encoding='utf-8')
COMUNAS_VIVIENDAS=sql2df(BD,"SELECT * FROM VIVIENDAS WHERE COMUNA=8101 OR COMUNA=8102 OR COMUNA=8103 OR COMUNA=8108 OR COMUNA= 8110 OR COMUNA=8112")
COMUNAS_VIVIENDAS.to_csv('BD_V_AMC.csv',encoding='utf-8')
```

Una vez nos aseguramos que pudimos leer y exportar lo que necesitamos del archivo en formato .db, podemos ahora empezar a leer y preparar la base de datos

Microdatos a nivel de personas

```python
PAMC=pd.read_csv(r'BD_P_AMC.csv') 
PAMC=PAMC.sort_values(by='ID_ZL_PER')
PAMC['ID_ZL_PER']=PAMC['ID_ZL_PER'].astype(str)
```
Microdatos a nivel de vivienda

```python
VIVAMC=pd.read_csv('BD_V_AMC.csv')
VIVAMC['ID_ZL_VIV']=VIVAMC['ID_ZL_VIV'].astype(str)
```
Se cargan los datos espaciales desde el .shp de de zonas urbanas del INE y se filtran por comuna

```python
Zonas=gpd.read_file(r'data\ZONA_C17.shp')

CAMC=gpd.GeoDataFrame(Zonas.loc[(Zonas['COMUNA']=='8101') |(Zonas['COMUNA']=='8102') | (Zonas['COMUNA']=='8103') | (Zonas['COMUNA']=='8108') | (Zonas['COMUNA']=='8110') | (Zonas['COMUNA']=='8112')])
```
Se introducen los poli y bicentros

```python
CBD=gpd.read_file(r'data\CBD\CBD.shp')
CBD_BI=gpd.read_file(r'data\CBD\CBD_bi.shp')
```

Intersección del .shp con geocodigos

```python
GCZ=CAMC.GEOCODIGO.astype(str).tolist()
GCPAMC=PAMC.ID_ZL_PER.astype(str).tolist()
GCVIVAMC=VIVAMC.ID_ZL_VIV.astype(str).tolist()
d=[GCZ,GCPAMC,GCVIVAMC]

INTER=list(reduce(set.intersection, [set(item) for item in d ]))

PAMC=PAMC[PAMC['ID_ZL_PER'].isin(INTER)]
VIVAMC=VIVAMC[VIVAMC['ID_ZL_VIV'].isin(INTER)]
CAMC=CAMC[CAMC['GEOCODIGO'].isin(INTER)]
CAMC.crs=Zonas.crs
```

> Una vez teniendo arreglado lo anterior, podemos empezar construir la base de datos

# Tasa de inmigración intrametropolitana

```python
MIG=PAMC.copy()
MIG.loc[MIG['P11'] == 1, ['P11']] = 0
MIG.loc[MIG['P11'] == 2, ['P11']] = 1
MIG.loc[MIG['P11'] == 3, ['P11']] = 2
MIG.loc[MIG['P11'] == 4, ['P11']] = 2
MIG.loc[MIG['P11'] == 5, ['P11']] = 2
MIG.loc[MIG['P11'] == 6, ['P11']] = 2
MIG.loc[MIG['P11'] == 7, ['P11']] = 2
MIG.loc[MIG['P11'] == 8, ['P11']] = 2
MIG.loc[MIG['P11'] == 9, ['P11']] = 2
MIG.loc[MIG['P11'] == 99, ['P11']] = None

TMI1=[]
TMI2=[]
ID=INTER
for i in range(len(ID)):
    GC=ID[i]
    COD_UNI=MIG.loc[MIG['ID_ZL_PER']==GC]
    q=0
    p=0
    for j in range(COD_UNI.shape[0]):
        if COD_UNI.iloc[j,17]==1: #En esta comuna
            q=q+1
        if COD_UNI.iloc[j,17]==2: #En otra comuna
            p=p+1
    TMI1.append(q)
    TMI2.append(p)
TMI1=pd.Series(TMI1) #En esta comuna
TMI2=pd.Series(TMI2) # En otra comuna
TII=TMI2/TMI1
```

# Porcentaje Jefes de hogar con escolaridad menor a 12 años

```python
JH=PAMC[(PAMC['P07']==1)]
JH=JH.drop(JH[(JH['ESCOLARIDAD']==99) | (JH['ESCOLARIDAD']==99)].index)
EJH14=[]
TPER=[]
ID=INTER
for i in range(len(ID)):
    GC = ID[i]
    COD_UNI = JH.loc[MIG['ID_ZL_PER'] == GC]
    q=0
    p=0
    for j in range(COD_UNI.shape[0]):
        p=p+1
        if COD_UNI.iloc[j, 41] < 12:
            q=q+1
    EJH14.append(q)
    TPER.append(p)
EJH14=pd.Series(EJH14)
TPER=pd.Series(TPER)
PEJH14=EJH14/TPER
```

# Cantidad de personas por zona (posterior densidad)

```python
CP=[]
ID=INTER
for i in range(len(ID)):
    GC = ID[i]
    COD_UNI = PAMC.loc[PAMC['ID_ZL_PER'] == GC]
    q=0
    for j in range(COD_UNI.shape[0]):
        q=q+1
    CP.append(q)
CP=pd.Series(CP)
```

# Porcentaje hacinamiento zonal

```python
VIVAMC_MP=VIVAMC[(VIVAMC['P02']==1) & (VIVAMC['P02']==1) != 99] #CON MORADORES PRESENTES P02==1
VIVAMC_MP.loc[VIVAMC['P04']==0,['P04']]=1 #Las 599 viviendas con 0 habitaciones exclusivas de dormitorio, fueron consideradas con 1
VIVAMC_MP = VIVAMC_MP.drop(VIVAMC_MP[VIVAMC_MP['CANT_PER']>=200].index) #Se eliminan las anonimizadas
HAC=VIVAMC_MP.CANT_PER/VIVAMC_MP.P04
VIVAMC_MP['HAC']=HAC

HAC_ZONAL=[]
TOT_VIV=[]
ID=INTER
for i in range(len(ID)):
    GC = ID[i]
    COD_UNI = VIVAMC_MP.loc[VIVAMC_MP['ID_ZL_VIV'] == GC]
    CH=0
    TV=0
    for j in range(COD_UNI.shape[0]):
        if COD_UNI.iloc[j, 20] >=2.4:
            CH=CH+1
        TV=TV+1
    TOT_VIV.append(TV)
    HAC_ZONAL.append(CH)

HAC_ZONAL=pd.Series(HAC_ZONAL)
TOT_VIV=pd.Series(TOT_VIV)
PHZ=HAC_ZONAL/TOT_VIV
```

# Agregar Variables al Dataframe
```python
BD=pd.DataFrame({'ID_ZL_PER':INTER})
BD['TII']=TII
BD['PEJH14']=PEJH12
BD['CP']=CP 
BD['PHZ']=PHZ
```

# Cruce de tablas para espacialización
```python
BD=CAMC.merge(BD,left_on='GEOCODIGO',right_on='ID_ZL_PER')
BD.crs=Zonas.crs
```
# Distancia de CBD a centroide más cercano

```python
EPSG=32719

CBD_proj=CBD.copy()
CBD_proj=CBD.to_crs(epsg=EPSG)  # Reproyeccion

CBD_BI=CBD_BI.copy()
CBD_BI_proj=CBD_BI.to_crs(epsg=EPSG)

BD_proj=BD.copy()
BD_proj=BD_proj.to_crs(epsg=EPSG)  # Reproyeccion
BD_proj['geometry']=BD_proj.centroid # Cambio de geometria a centroides
```

# Para policentros

```python
from shapely.ops import nearest_points
from shapely.geometry import Point
DIST1=[]
for i in range(len(BD.ID_ZL_PER)):
    OD=nearest_points(Point(BD_proj.geometry[i]),CBD_proj.geometry.unary_union) #0.- origen , 1.- destino, formato tupla; pero no sabes cual sale primero
    ZL=Point(OD[0])
    Nearest_CBD=Point(OD[1])
    dist=ZL.distance(Nearest_CBD)
    DIST1.append(dist)
```

# Para bicentros

```python
from shapely.ops import nearest_points
from shapely.geometry import Point
DIST2=[]
for i in range(len(BD.ID_ZL_PER)):
    OD=nearest_points(Point(BD_proj.geometry[i]),CBD_BI_proj.geometry.unary_union) #1.- origen , 2.- destino, formato tupla
    ZL=Point(OD[0])
    Nearest_CBD=Point(OD[1])
    dist=ZL.distance(Nearest_CBD)
    DIST2.append(dist)
```

# Cambio a polígonos

```python
BD_proj['geometry']=BD['geometry'].to_crs(epsg=EPSG) #cambiamos a polígono no proyectado
BD_proj['DPOLY']=DIST1
BD_proj['DBI']=DIST2
BD_proj['DPOLY']=BD_proj['DPOLY']/1000
BD_proj['DBI']=BD_proj['DBI']/1000
```

# Cálculo de áreas

```python
BD_proj['AREA']=BD_proj.geometry.area/10000 #en metros hectareas
```
# Densidad

```python
BD_proj['Densidad']=BD_proj.CP/BD_proj.AREA
```

# Exportar a shp

```python
BD_proj.to_file(r'tu dirección')
```