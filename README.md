# Guardería Gaghiel - Parcial SQL 
## Parcial AD - Tema A 
### AD.A1 
#### Enunciado 
AD.A1 -  Reforma de solicitudes de contrato  . La empresa  ha detectado 
que algunas solicitudes de contratos se deberían hacer a nombre de dos 
clientes o más, sin embargo el sistema actual solo permite registrar 
1. Por este motivo debemos modificar la estructura de la base de datos 
para permitir que las solicitudes sean de muchos clientes y los 
clientes puedan hacer muchas solicitudes. Modificar la estructura de 
la base de datos para el nuevo modelo, migrar el único cliente actual 
al nuevo modelo en una transacción y borrar la estructura anterior.
<img width="640" height="70" alt="image" src="https://github.com/user-attachments/assets/ac7305ca-5859-46b3-9348-2d5ff1e7afa0" />

#### Resolución A
```sql 
create table cliente_contrato ( 
id_solicitud int unsigned not null, 
id_cliente int unsigned  not null, 
primary key (id_solicitud, id_cliente), 
constraint fk_cliente_contrato_solicitud_contrato 
foreign key (id_solicitud) references solicitud_contrato(id) 
on delete restrict on update cascade, 
constraint fk_cliente_contrato_persona 
foreign key (id_cliente) references persona(id) 
on delete restrict on update cascade 
) engine=InnoDB; 
begin; 
insert into cliente_contrato 
select id, id_cliente 
from solicitud_contrato; 
commit; 
alter table solicitud_contrato 
drop constraint fk_solicitud_contrato_persona, 
drop column id_cliente; 
```
#### Resolución B - La Mia
```sql 
CREATE TABLE `inmobiliaria_calciferhowl`.`cliente_contrato` (
  `id_cliente` INT UNSIGNED NOT NULL,
  `id_solicitud` INT UNSIGNED NOT NULL,
  PRIMARY KEY (`id_cliente`, `id_solicitud`),
  INDEX `fk_cliente_contrato_contrato_idx` (`id_solicitud` ASC) VISIBLE,
  CONSTRAINT `fk_cliente_contrato_cliente`
    FOREIGN KEY (`id_cliente`)
    REFERENCES `inmobiliaria_calciferhowl`.`persona` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION,
  CONSTRAINT `fk_cliente_contrato_contrato`
    FOREIGN KEY (`id_solicitud`)
    REFERENCES `inmobiliaria_calciferhowl`.`solicitud_contrato` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION);

 START TRANSACTION;

 INSERT INTO cliente_contrato(id_cliente,id_solicitud)
 SELECT sol.id_cliente,sol.id
 FROM solicitud_contrato sol;
 
 COMMIT;
 
ALTER TABLE `inmobiliaria_calciferhowl`.`solicitud_contrato` 
DROP FOREIGN KEY `fk_solicitud_contrato_persona`;
ALTER TABLE `inmobiliaria_calciferhowl`.`solicitud_contrato` 
DROP COLUMN `id_cliente`,
DROP INDEX `fk_solicitud_contrato_persona_idx` ;
;

``` 
### AD.A2 
#### Enunciado 
AD.A2 -  Oportunidades perdidas  . Listar las solicitudes  con estado 
rechazada cuyo importe mensual pactado sea mayor o igual al 70% del 
valor de la propiedad a la fecha de dicha solicitud de contrato 
(fecha_solicitud) y que a pesar de haber sido rechazada la solicitud 
no tengan ninguna garantía rechazada. Indicar id y fecha de solicitud; 
id, nombre y apellido del agente y proporción entre el importe mensual 
y el valor a esa fecha. 
Nota: Los estados de solicitudes de contrato pueden ser “en proceso”, 
“en alquiler” o “rechazada”. 
#### Resolución 
```sql 
select sc.id id_solicitud, sc.fecha_solicitud, age.id id_agente, age.nombre, age.apellido, sc.importe_mensual/vp.valor proporcion 
from solicitud_contrato sc 
inner join valor_propiedad vp on sc.id_propiedad=vp.id_propiedad and vp.fecha_hora_desde=( 
select max(ult_val.fecha_hora_desde) 
from valor_propiedad ult_val 
where ult_val.id_propiedad=sc.id_propiedad 
and ult_val.fecha_hora_desde<=sc.fecha_solicitud 
) 
inner join persona age on sc.id_agente=age.id 
left join garantia g on sc.id=g.id_solicitud and g.estado='rechazada' 
where sc.estado = 'rechazada' and sc.importe_mensual/vp.valor >=0.7 and g.id_solicitud is null; 
#el left equivale a agregar al where el not in 
# sc.id not in (select g.id_solicitud from garantia g where 
g.estado='rechazada') 
``` 
#### Resolución B 
```sql 
drop temporary table if exists ultima; 
create temporary table ultima (select 
sc.id,vp.id_propiedad,max(vp.fecha_hora_desde) ultima_fecha 
from valor_propiedad vp 
inner join solicitud_contrato sc on vp.id_propiedad= sc.id_propiedad 
where vp.fecha_hora_desde <= sc.fecha_solicitud 
group by  sc.id, vp.id_propiedad); 
/*select * from ultima;*/ 
Select sc.id, sc.fecha_solicitud, concat(per.nombre,"   ", 
per.apellido),sc.importe_mensual/vp.valor 
from valor_propiedad vp 
inner join ultima ult on ult.id_propiedad=vp.id_propiedad and 
ult.ultima_fecha= vp.fecha_hora_desde 
inner join solicitud_contrato sc on sc.id_propiedad=ult.id_propiedad 
and sc.fecha_solicitud>=ult.ultima_fecha 
inner join persona per on sc.id_agente= per.id 
where sc.importe_mensual/vp.valor >= 0.7 and sc.estado= 'rechazada' 
and sc.id not in (select id_solicitud from garantia where estado 
='rechazada') ; 
```
#### Resolución C - La Mia 
```sql 
WITH precio_vigente_solicitud AS (
    -- CORRECCIÓN: Buscamos la fecha de precio específica para CADA solicitud (sc.id), 
    SELECT sc.id AS id_solicitud,
        MAX(vp.fecha_hora_desde) AS fecha_vigencia
    FROM solicitud_contrato sc
    INNER JOIN valor_propiedad vp ON sc.id_propiedad = vp.id_propiedad 
    AND vp.fecha_hora_desde <= sc.fecha_solicitud 
    WHERE sc.estado = 'rechazada'
    GROUP BY sc.id
)
SELECT 
    sc.id AS id_solicitud, sc.fecha_solicitud, ag.id AS id_agente, ag.nombre AS nombre_agente, 
    ag.apellido AS apellido_agente, (sc.importe_mensual / vp.valor) AS proporcion
FROM solicitud_contrato sc
INNER JOIN precio_vigente_solicitud pvs ON sc.id = pvs.id_solicitud
INNER JOIN valor_propiedad vp ON vp.id_propiedad = sc.id_propiedad AND vp.fecha_hora_desde = pvs.fecha_vigencia
INNER JOIN persona ag ON ag.id = sc.id_agente
WHERE 
    sc.estado = 'rechazada'
    AND (sc.importe_mensual / vp.valor) >= 0.70
    AND sc.id NOT IN (
        SELECT gar.id_solicitud 
        FROM garantia gar
        WHERE gar.estado = 'rechazada'
    );
``` 
### AD.A3 
#### Enunciado 
AD.A3 -  Trending props  . En base a las visitas de 2025,  listar la 
propiedad más popular de cada mes. Se considera la propiedad más 
popular de cada mes a la que tenga mayor cantidad de visitas el mismo 
mes en base a la fecha de visita. En cada mes de 2025 se debe indicar 
la propiedad más popular (o las más populares si tuvieran la misma 
cantidad de visitas) l y para dicho mes indicar los datos de la última 
visita  de la propiedad más popular  . Indicar mes; id, zona y tipo de 
propiedad, cantidad total de visitas; fecha de la última visita  (para 
dicho mes y propiedad)  y del agente de dicha visita indicar su id, 
nombre y apellido. Si en un mes de 2025 no hubiera visitas no debe 
mostrarse. 
Nota: la función month devuelve el mes de una fecha 
#### Resolución 
```sql 
with visitas_mes as ( 
select month(v.fecha_hora_visita) mes , v.id_propiedad, p.tipo, p.zona, count(*) cant_visitas,
max(v.fecha_hora_visita) ult_visita 
# si no se dan cuenta acá del max 
# deben hacer una o dos subonconsulta / TT / CTE más 
# para calcularlo, sería correcto en ese caso 
from visita v 
inner join propiedad p on v.id_propiedad=p.id 
where year(fecha_hora_visita)=2025 
group by month(v.fecha_hora_visita), v.id_propiedad, p.tipo, p.zona 
), max_vis as ( 
select mes, max(cant_visitas) max_vis 
from visitas_mes 
group by mes 
)
select vmes.mes, vmes.id_propiedad, vmes.zona, vmes.tipo, vmes.cant_visitas, vmes.ult_visita, age.id id_agente, age.nombre nombre_agente, age.apellido 
apellido_agente 
from visitas_mes vmes 
inner join max_vis  maxv on vmes.mes=maxv.mes and vmes.cant_visitas=maxv.max_vis 
inner join visita vi#  on vmes.mes=month(vi.fecha_hora_visita) on vmes.id_propiedad=vi.id_propiedad and vmes.ult_visita=vi.fecha_hora_visita 
inner join persona age on vi.id_agente=age.id 
where year(vi.fecha_hora_visita)=2025; 
# probablmente no se den cuenta del where o lo pongan en on 
# no descontaría por esto; 
``` 
#### Resolución B 
```sql 
drop temporary table if exists cant_visitas_mensuales; 
create temporary table cant_visitas_mensuales (select id_propiedad, 
month(fecha_hora_visita) mes , max(fecha_hora_visita) ultima_visita, 
count(*) as cantidad_visitas 
from visita 
where year(fecha_hora_visita) = 2025 
group by id_propiedad, month(fecha_hora_visita)); 
drop temporary table if exists max_mensual; 
create temporary table max_mensual ( 
select mes, max(cantidad_visitas) cant_mensual 
from cant_visitas_mensuales 
group by mes); 
select cvm.mes,cvm.id_propiedad, pro.zona,pro.tipo, 
cvm.cantidad_visitas, cvm.ultima_visita, vis.id_agente, concat ( 
per.nombre , "  ",per.apellido) 
from cant_visitas_mensuales cvm 
inner join max_mensual maxm on  cvm.mes=maxm.mes and 
cvm.cantidad_visitas=maxm.cant_mensual 
inner join propiedad pro on cvm.id_propiedad=pro.id 
inner join visita vis on cvm.id_propiedad=vis.id_propiedad and 
vis.fecha_hora_visita=cvm.ultima_visita 
inner join persona per on per.id=vis.id_agente;
```
#### Resolución C - La Mia 
```sql 
WITH metricas_propiedad_mes AS (
    -- 1. Calculamos todo lo referente a la propiedad en ese mes: 
    -- Cuántas veces se visitó y cuándo fue la última vez.
    SELECT vis.id_propiedad, MONTH(vis.fecha_hora_visita) AS mes, COUNT(*) AS cantidad_visitas,
        MAX(vis.fecha_hora_visita) AS fecha_ultima_visita
    FROM visita vis
    WHERE YEAR(vis.fecha_hora_visita) = 2025
    GROUP BY vis.id_propiedad, MONTH(vis.fecha_hora_visita)
), 
max_visitas_por_mes AS (
    -- 2. Definimos la vara alta: ¿Cuál fue el número máximo de visitas logrado en cada mes?
    SELECT mpm.mes, MAX(mpm.cantidad_visitas) AS max_cantidad
    FROM metricas_propiedad_mes mpm
    GROUP BY mpm.mes
)
SELECT mpm.mes, p.id AS id_propiedad, p.zona, p.tipo, mpm.cantidad_visitas, mpm.fecha_ultima_visita, 
    ag.id AS id_agente, ag.nombre AS nombre_agente, ag.apellido AS apellido_agente
FROM metricas_propiedad_mes mpm
-- 1. Filtramos: Nos quedamos solo con las propiedades que igualaron el récord del mes
INNER JOIN max_visitas_por_mes maxv ON mpm.mes = maxv.mes AND mpm.cantidad_visitas = maxv.max_cantidad
-- 2. Datos de la propiedad
INNER JOIN propiedad p ON p.id = mpm.id_propiedad
-- 3. Para obtener al agente, hacemos join con la visita específica
-- Usando la clave compuesta: propiedad + fecha exacta de la última visita
INNER JOIN visita v_detalle ON v_detalle.id_propiedad = mpm.id_propiedad 
							AND v_detalle.fecha_hora_visita = mpm.fecha_ultima_visita
INNER JOIN persona ag ON ag.id = v_detalle.id_agente
ORDER BY mpm.mes;
```
### AD.A4 
#### Enunciado 
AD.A4 -  Mejoras en las ofertas  . La empresa requiere  un listado de las 
propiedades cuya cantidad de solicitudes de contrato en 2024 haya 
aumentado con respecto al promedio de cantidad de solicitudes de 
contrato de las propiedades del mismo tipo en 2023. Si una propiedad 
no tuvo solicitudes debería tenerse en cuenta como 0 para el promedio. 
Indicar id, tipo, zona, situación de la propiedad, cantidad de 
solicitudes en 2024 y promedio de solicitudes del tipo en 2023. 
#### Resolución 
```sql 
with  sc_prop_tipo_anio as ( 
select p.id id_propiedad, p.tipo, p.zona, p.situacion, year(sc.fecha_solicitud) anio, count(sc.fecha_solicitud) cant_sol 
from propiedad p 
left join solicitud_contrato sc on sc.id_propiedad=p.id 
# left, count(atrib) y coalesce son para 
# calcular bien el promedio aún si 
# una prop no tiene solicitudes en 2024 
group by p.id, p.tipo, p.zona, p.situacion, year(sc.fecha_solicitud) 
), sctipo_2023 as ( 
select scpro.tipo, avg(cant_sol) prom_sol 
from sc_prop_tipo_anio scpro 
where scpro.anio=2023 
group by scpro.tipo 
) 
select scpta.id_propiedad, scpta.tipo, scpta.zona, scpta.situacion, scpta.cant_sol cant_sol_2024, coalesce(sctipo_2023.prom_sol,0) prom_sol_2023 
from sc_prop_tipo_anio scpta 
left join sctipo_2023 on scpta.tipo=sctipo_2023.tipo 
where scpta.anio=2024 and scpta.cant_sol > coalesce(sctipo_2023.prom_sol,0); 
``` 
#### Resolución B 
```sql 
drop temporary table if exists cantidad_2023; 
create temporary table cantidad_2023
select pro.id,pro.tipo, count(sc.id_propiedad) cant_solicitudes 
from propiedad pro 
left join solicitud_contrato sc on  pro.id=sc.id_propiedad  and year(sc.fecha_solicitud)=2023 
group by pro.id, pro.tipo;

drop temporary table if exists promedioxtipo; 
create temporary table promedioxtipo
select c23.tipo, avg(c23.cant_solicitudes) promedio 
from cantidad_2023 c23 
group by c23.tipo;

drop temporary table if exists cantidad_2024; 
create temporary table cantidad_2024
select 
pro.id,pro.tipo,pro.zona,pro.situacion, count(sc.id_propiedad) cant_solicitudes 
from propiedad pro 
left join solicitud_contrato sc on  pro.id=sc.id_propiedad  and year(sc.fecha_solicitud)=2024 
group by pro.id, pro.tipo, pro.zona,pro.situacion;

select c24.id,  c24.tipo, c24.zona, c24.situacion, c24.cant_solicitudes, prot.promedio 
from cantidad_2024 c24 
inner join   promedioxtipo prot on c24.tipo=prot.tipo 
where c24.cant_solicitudes > prot.promedio;
```
#### Resolución C - La Mia
```sql 
/*
AD.A4 - Mejoras en las ofertas . 
La empresa requiere un listado de las propiedades cuya cantidad de solicitudes de contrato en 2024 haya aumentado 
con respecto al promedio de cantidad de solicitudes de contrato de las propiedades del mismo tipo en 2023. 
Si una propiedad no tuvo solicitudes debería tenerse en cuenta como 0 para el promedio.
Indicar id, tipo, zona, situación de la propiedad, cantidad de solicitudes en 2024 y promedio de solicitudes del
tipo en 2023.
*/

WITH cant_sol_2023 AS (
SELECT pdad.id as id_propiedad, count(fecha_solicitud) cant_sol
FROM propiedad pdad
LEFT JOIN solicitud_contrato sol on sol.id_propiedad = pdad.id AND YEAR(sol.fecha_solicitud)=2023
GROUP BY pdad.id
), prom_2023 AS (
SELECT avg(cs.cant_sol) prom, pdad.tipo
from cant_sol_2023 cs
INNER JOIN propiedad pdad ON pdad.id=cs.id_propiedad
GROUP BY pdad.tipo
), cant_sol_2024 AS (
SELECT pdad.id as id_propiedad, count(fecha_solicitud) cant_sol
FROM propiedad pdad
LEFT JOIN solicitud_contrato sol on sol.id_propiedad = pdad.id AND YEAR(sol.fecha_solicitud)=2024
GROUP BY pdad.id
)
SELECT pdad.id, pdad.zona, pdad.tipo, pdad.situacion, cs2024.cant_sol, prom_2023.prom
FROM propiedad pdad
INNER JOIN cant_sol_2024 cs2024 ON cs2024.id_propiedad=pdad.id
INNER JOIN prom_2023  on prom_2023.tipo=pdad.tipo
WHERE cs2024.cant_sol > prom_2023.prom
ORDER BY pdad.id
```
## Parcial AD - Tema B 
### AD.B1 
#### Enunciado 
AD.B1 - La inmobiliaria nos informa que en algunas solicitudes de 
contratos se necesita registrar varios agentes que cooperaron durante 
un contrato. Una solicitud de contrato puede involucrar a varios 
agentes asignados y un agente asignado puede participar en muchas 
solicitudes. Se requiere modificar la estructura de la base de datos 
para soportar este cambio en el modelo, migrar el agente asignado 
registrado en las solicitudes existentes al nuevo modelo en una 
transacción y eliminar la estructura anterior. 
<img width="636" height="327" alt="image" src="https://github.com/user-attachments/assets/b5fceed3-48fb-47d9-a26b-6315ab830c2c" />

#### Resolución 
```sql 
create table agente_solicitud ( 
id_solicitud int unsigned not null, 
id_agente int unsigned not null, 
id_propiedad int unsigned not null, 
fecha_hora_desde datetime not null, 
primary key (id_solicitud, id_agente, id_propiedad, 
fecha_hora_desde), 
constraint fk_agente_solicitud_solicitud_contrato 
foreign key (id_solicitud) 
references solicitud_contrato(id) 
on delete restrict on update cascade, 
constraint fk_agente_solicitud_agente_asignado 
foreign key (id_agente, id_propiedad, fecha_hora_desde) 
references agente_asignado(id_agente, id_propiedad, 
fecha_hora_desde) 
on delete restrict on update cascade 
) engine = InnoDB; 
begin; 
insert into agente_solicitud 
select id, id_agente, id_propiedad, fecha_hora_desde 
from solicitud_contrato; 
commit; 
alter table solicitud_contrato 
drop constraint fk_solicitud_contrato_agente_asignado, 
drop column id_agente, 
drop column id_propiedad, 
drop column fecha_hora_desde; 
```
#### Resolución B - La Mia
```sql 
CREATE TABLE `inmobiliaria_calciferhowl`.`agente_solicitud` (
  `id_agente` INT UNSIGNED NOT NULL,
  `id_propiedad` INT UNSIGNED NOT NULL,
  `fecha_hora_desde` DATETIME NOT NULL,
  `id_solicitud` INT UNSIGNED NOT NULL,
  PRIMARY KEY (`id_agente`, `id_propiedad`, `id_solicitud`,`fecha_hora_desde` ),
  INDEX `fk_agente_solicitud_solicitud_contrato_idx` (`id_solicitud` ASC) VISIBLE,
  INDEX `fk_agente_solicitud_agente_asignado_idx` (`id_agente` ASC, `id_propiedad` ASC, `fecha_hora_desde` ASC) VISIBLE,
  CONSTRAINT `fk_agente_solicitud_solicitud_contrato`
    FOREIGN KEY (`id_solicitud`)
    REFERENCES `inmobiliaria_calciferhowl`.`solicitud_contrato` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION,
  CONSTRAINT `fk_agente_solicitud_agente_asignado`
    FOREIGN KEY (`id_agente` , `id_propiedad` , `fecha_hora_desde`)
    REFERENCES `inmobiliaria_calciferhowl`.`agente_asignado` (`id_agente` , `id_propiedad` , `fecha_hora_desde`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION);
    
START TRANSACTION;

INSERT INTO agente_solicitud (id_agente,id_propiedad,fecha_hora_desde,id_solicitud)
SELECT soli.id_agente, soli.id_propiedad, soli.fecha_hora_desde, soli.id
FROM solicitud_contrato soli
;

COMMIT;

ALTER TABLE `inmobiliaria_calciferhowl`.`solicitud_contrato` 
DROP FOREIGN KEY `fk_solicitud_contrato_agente_asignado`;
ALTER TABLE `inmobiliaria_calciferhowl`.`solicitud_contrato` 
DROP COLUMN `fecha_hora_desde`,
DROP COLUMN `id_propiedad`,
DROP COLUMN `id_agente`,
DROP INDEX `fk_solicitud_contrato_agente_asignado_idx` ;
;
``` 
### AD.B2 
#### Enunciado 
AD.B2 -  Los clavos  . La inmobiliaria quiere identificar  las propiedades 
que pagan alquileres bajos pero no están siendo mostradas. Se requiere 
listar los pagos en concepto de alquiler cuyo importe sea menor al 70% 
del valor de la propiedad a la fecha de pago del alquiler y no tengan 
visitas posteriores a la fecha de contrato de la solicitud 
correspondiente. Indicando id y fecha de contrato de la solicitud; 
importe y fecha y hora de pago, proporción entre importe pagado y 
valor de la propiedad a la fecha de pago; id, tipo, zona y dirección 
de la propiedad. 
Nota: Los conceptos de pago pueden ser “comision”, “deposito”, 
“sellado” o “pago alquiler”. 
#### Resolución 
```sql 
with sol_ult_val as ( 
select p.id_solicitud, p.fecha_hora_pago, p.importe, sc.id_propiedad, sc.fecha_contrato, p.concepto, max(vp.fecha_hora_desde) ult_fec 
from pago p 
inner join solicitud_contrato sc on p.id_solicitud=sc.id 
inner join valor_propiedad vp on vp.id_propiedad=sc.id_propiedad and vp.fecha_hora_desde <=p.fecha_hora_pago 
where concepto='pago alquiler' 
group by p.id_solicitud, p.fecha_hora_pago, p.importe, 
sc.id_propiedad 
) 
select suv.id_solicitud, suv.fecha_contrato, suv.fecha_hora_pago, suv.ult_fec, suv.importe/vp.valor,
prop.id id_propiedad, prop.tipo, prop.zona, prop.direccion 
from sol_ult_val suv 
inner join valor_propiedad vp on suv.id_propiedad=vp.id_propiedad and suv.ult_fec=vp.fecha_hora_desde 
inner join propiedad prop on suv.id_propiedad=prop.id 
where suv.importe/vp.valor < 0.7 
and  suv.id_propiedad not in ( 
select v.id_propiedad 
from visita v 
where v.fecha_hora_visita>=suv.fecha_contrato 
); 
```
#### Resolución B - La Mia
```sql 
WITH valor_prop_necesario AS (
    -- Necesitamos saber qué valor tenía la propiedad en el momento EXACTO de CADA pago.
    SELECT sol.id AS id_solicitud, pa.fecha_hora_pago, MAX(vp.fecha_hora_desde) AS ult_fecha_valor
    FROM valor_propiedad vp
    INNER JOIN solicitud_contrato sol ON sol.id_propiedad = vp.id_propiedad
    INNER JOIN pago pa ON pa.id_solicitud = sol.id AND vp.fecha_hora_desde <= pa.fecha_hora_pago
    WHERE sol.fecha_contrato IS NOT NULL AND pa.concepto = 'pago alquiler'
    GROUP BY sol.id, pa.fecha_hora_pago
)
SELECT sol.id AS id_solicitud, sol.fecha_contrato, pa.importe AS importe_pago, pa.fecha_hora_pago,
    (pa.importe / vp.valor) AS proporcion, pdad.id AS id_propiedad, pdad.tipo, pdad.zona, pdad.direccion
FROM valor_prop_necesario vpn
INNER JOIN solicitud_contrato sol ON sol.id = vpn.id_solicitud
INNER JOIN propiedad pdad ON pdad.id = sol.id_propiedad
INNER JOIN pago pa ON pa.id_solicitud = sol.id AND pa.fecha_hora_pago = vpn.fecha_hora_pago 
INNER JOIN valor_propiedad vp ON sol.id_propiedad = vp.id_propiedad AND vp.fecha_hora_desde = vpn.ult_fecha_valor
WHERE (pa.importe / vp.valor) < 0.70 AND sol.id_propiedad NOT IN (
        SELECT vis.id_propiedad
        FROM visita vis
        WHERE vis.fecha_hora_visita > sol.fecha_contrato
    );
``` 
### AD.B3 
#### Enunciado 
AD.B3 -  Agente del año  . En base a las solicitudes  con contrato, listar 
para cada año quien fue el mejor agente. El mejor agente de un año es 
aquel cuya, suma de importes mensuales de todas las solicitudes que 
gestionó y se hayan contratado ese año, sea la más alta (en base a la 
fecha de contrato de la solicitud de contrato). En cada año se debe 
indicar el mejor agente (o los mejores si vendieran la misma cantidad) 
y para ese año indicar los datos de la mejor de las solicitudes de 
dicho agente (la que tenga importe más alto). Indicar año; id, nombre 
y apellido del agente, total de importes mensuales; id, fecha de 
solicitud, importe mensual y fecha de contrato de la mejor solicitud; 
id, nombre y apellido del cliente de dicha solicitud. Si un año no 
hubiera solicitudes con contrato no debe mostrarse 
Nota: la función year devuelve el año de una fecha 
#### Resolución 
```sql 
with tot_ag as ( 
select year(sc.fecha_contrato) anio, sc.id_agente, ag.nombre nombre_agente, ag.apellido apellido_agente,
sum(sc.importe_mensual) total_importes, max(sc.importe_mensual) mejor_imp 
# si no se dan cuenta acá del max 
# deben hacer una o dos subonconsulta / TT / CTE más 
# para calcularlo, sería correcto en ese caso 
from solicitud_contrato sc 
inner join persona ag 
on sc.id_agente=ag.id 
where sc.fecha_contrato is not null 
group by year(sc.fecha_contrato), sc.id_agente, ag.nombre, ag.apellido 
), max_anio as ( 
select anio, max(total_importes) max_anio 
from tot_ag 
group by anio 
) 
select ta.anio, ta.id_agente, ta.nombre_agente, ta.apellido_agente, ta.total_importes,sc.id, sc.fecha_solicitud,
sc.importe_mensual, sc.fecha_contrato, cli.id id_cliente, cli.nombre nombre_cliente, cli.apellido apellido_cliente 
from tot_ag ta 
inner join max_anio ma on ta.anio=ma.anio and ta.total_importes=ma.max_anio 
inner join solicitud_contrato sc on ta.id_agente=sc.id_agente and year(sc.fecha_contrato)=ta.anio 
and sc.importe_mensual=ta.mejor_imp 
inner join persona cli # puede obtenerse acá o en la primer CTE 
on sc.id_cliente=cli.id 
where sc.fecha_contrato is not null 
# probablmente no se den cuenta del where o lo pongan en on 
# no descontaría por esto; 
``` 
#### Resolución B 
```sql 
drop temporary table if exists tot_anual; 
create temporary table tot_anual (
select id_agente, year(fecha_contrato) anio , max(importe_mensual) mejor_importe, sum(importe_mensual) suma_importe 
from solicitud_contrato 
where year(fecha_solicitud) = year(fecha_contrato) 
group by id_agente, year(fecha_contrato));

drop temporary table if exists max_anual; 
create temporary table max_anual ( 
select anio, max(suma_importe) mejor 
from tot_anual 
group by anio); 
select tta.anio,tta.id_agente, tta.suma_importe, concat ( age.nombre , "  " ,age.apellido) , sc.id,
sc.fecha_contrato, sc.importe_mensual, sc.fecha_contrato, cli.id, concat ( cli.nombre , "  ",cli.apellido) 
from tot_anual tta 
inner join max_anual maxa on  tta.anio=maxa.anio and tta.suma_importe=maxa.mejor 
inner join persona age on tta.id_agente=age.id 
inner join solicitud_contrato sc on tta.id_agente=sc.id_agente and sc.importe_mensual=tta.mejor_importe 
inner join persona cli on sc.id_cliente=cli.id 
where sc.fecha_contrato is not null; 
```
#### Resolución C - Personal
```sql 
WITH metricas_agente AS (
    -- Reemplaza a tus CTEs 'max_pago' y 'suma_importe'.
    -- Calculamos en un solo paso el total vendido y la mejor venta individual por agente/año.
    SELECT 
        sol.id_agente,
        YEAR(sol.fecha_contrato) AS anio,
        SUM(sol.importe_mensual) AS sum_importe, -- Total acumulado para competir por el año
        MAX(sol.importe_mensual) AS max_importe  -- Dato para buscar la mejor solicitud luego
    FROM solicitud_contrato sol
    WHERE sol.fecha_contrato IS NOT NULL
    GROUP BY sol.id_agente, YEAR(sol.fecha_contrato)
), 
max_importe_anio AS (
    -- Tu estructura original se mantiene: buscamos la vara más alta de cada año.
    SELECT 
        ma.anio,
        MAX(ma.sum_importe) AS max_venta_global_anio
    FROM metricas_agente ma
    GROUP BY ma.anio
)
SELECT 
    ma.anio, ag.id AS id_agente, ag.nombre AS nombre_agente, ag.apellido AS apellido_agente,
    ma.sum_importe AS total_ventas_anio, sol.id AS id_solicitud, sol.fecha_solicitud, sol.importe_mensual,
    sol.fecha_contrato, cli.id AS id_cliente, cli.nombre AS nombre_cliente, cli.apellido AS apellido_cliente
FROM metricas_agente ma
-- 1. Filtramos solo los agentes que alcanzaron el máximo global del año
INNER JOIN max_importe_anio mia 
    ON ma.anio = mia.anio 
    AND ma.sum_importe = mia.max_venta_global_anio
-- 2. Buscamos los datos del agente
INNER JOIN persona ag 
    ON ag.id = ma.id_agente
-- 3. Buscamos la solicitud específica que coincide con el 'max_importe' de ese agente en ese año
INNER JOIN solicitud_contrato sol 
    ON sol.id_agente = ma.id_agente 
    AND YEAR(sol.fecha_contrato) = ma.anio
    AND sol.importe_mensual = ma.max_importe -- Aquí conectamos con la mejor venta individual
-- 4. Buscamos al cliente
INNER JOIN persona cli 
    ON cli.id = sol.id_cliente
``` 
### AD.B4 
#### Enunciado 
AD.B4 -  Tendencia de busqueda  . La empresa requiere  un listado de las 
propiedades cuya cantidad de visitas en 2025 supere al promedio de 
cantidad de visitas de cada propiedad del mismo tipo durante 2024. Si 
una propiedad no tiene visitas debería tenerse en cuenta como 0 para 
el promedio. Indicar id, tipo, zona y situación de la propiedad, 
cantidad de visitas en 2025 y promedio de visitas del tipo de 
propiedad en 2024. 
#### Resolución 
```sql 
with v_prop_tipo_anio as ( 
select p.id id_propiedad, p.tipo, p.zona, p.situacion,
year(vi.fecha_hora_visita) anio, count(vi.fecha_hora_visita) cant_vis 
from propiedad p 
left join visita vi  # left, count(atrib) y coalesce son para 
# calcular bien el promedio aún si 
# una prop no tiene visitas en 2024 
on vi.id_propiedad=p.id 
where vi.fecha_hora_visita >= '20240101' and  vi.fecha_hora_visita < '20260101' 
group by p.id, p.tipo, p.zona, p.situacion, year(vi.fecha_hora_visita) # si no se dan cuenta 
#y lo hacen en 2 temp/cte/subcons es aceptable 
), vtipo_2024 as ( # se puede hacer esto junto a las de 2025 sin cte 
# pero seguramente sea muy dificil para los alumnos 
select vipro.tipo, avg(vipro.cant_vis) prom_vis 
from v_prop_tipo_anio vipro 
where vipro.anio=2024 
group by vipro.tipo 
) 
select vpta.id_propiedad, vpta.tipo, vpta.zona, vpta.situacion, vpta.cant_vis cant_vis_2025,
coalesce(vtipo_2024.prom_vis,0) prom_vis_2024 
from v_prop_tipo_anio vpta 
left join vtipo_2024 on vpta.tipo=vtipo_2024.tipo 
where vpta.anio=2025 and vpta.cant_vis>coalesce(vtipo_2024.prom_vis,0); 
``` 
#### Resolución B 
```sql 
drop temporary table if exists cantidad_2024; 
create temporary table cantidad_2024 select pro.id,pro.tipo, 
count(v.id_propiedad) cant_vistas 
from propiedad pro 
left join visitas v on  pro.id=v.id_propiedad  and year(v.fecha_hora_visita)=2024 
group by pro.id, pro.tipo;

drop temporary table if exists promedioxtipo; 
create temporary table promedioxtipo select c24.tipo, 
avg(c24.cant_visitas) promedio 
from cantidad_2024 c24 
group by c24.tipo;

drop temporary table if exists cantidad_2025; 
create temporary table cantidad_2025 select 
pro.id,pro.tipo,pro.zona,pro.situacion, count(v.id_propiedad) cant_visitas 
from propiedad pro 
left join visitas  v on  pro.id=v.id_propiedad  and year(v.fecha_hora_visita)=2025 
group by pro.id, pro.tipo, pro.zona, pro.situacion; 
select c25.id,  c25.tipo, c25.zona, c25.situacion, c25.cant_visitas, 
prot.promedio 
from cantidad_2025 c25 
inner join  promedioxtipo prot on c25.tipo=prot.tipo 
where c25.cant_visitas > prot.promedio; 
``` 
#### Resolución C 
```sql 
with cant_2024 as ( 
select p.id id_propiedad, p.tipo, count(v.id_propiedad) cnt 
from propiedad p 
left join visita v 
on p.id = v.id_propiedad 
and year(fecha_hora_visita) = 2024 
group by  p.id, p.tipo 
) 
select p.id, p.tipo, p.zona, p.situacion, count(*) total_2025, (select 
avg(cnt) from cant_2024 where cant_2024.tipo = p.tipo) promedio_2024 
from visita v 
join propiedad p on v.id_propiedad = p.id 
where year(v.fecha_hora_visita) = 2025 
group by p.id, p.tipo, p.zona, p.situacion 
having count(*) > ( 
select avg(cnt) 
from cant_2024 
where cant_2024.tipo = p.tipo 
); 
```
#### Resolución D - La Mia
```sql 
WITH cant_visitas_2024 AS (
SELECT pd.id, count(vis.fecha_hora_visita) cant_visitas 
FROM propiedad pd
LEFT JOIN visita vis ON pd.id=vis.id_propiedad AND YEAR(vis.fecha_hora_visita)=2024
group by pd.id
), avg_visitas_2024 AS (
SELECT pd.tipo, avg(cant_visitas) avg_2024
from cant_visitas_2024 cv
INNER JOIN propiedad pd ON pd.id=cv.id
GROUP BY pd.tipo
), cant_visitas_2025 AS (
SELECT pd.id, count(vis.fecha_hora_visita) cant_visitas 
FROM propiedad pd
LEFT JOIN visita vis ON pd.id=vis.id_propiedad AND YEAR(vis.fecha_hora_visita)=2025
group by pd.id 
)

SELECT pd.id, pd.tipo, pd.zona, pd.situacion, avgold.avg_2024 as promedio_2024, cvnew.cant_visitas as visitas_2025
FROM propiedad pd
INNER JOIN avg_visitas_2024 avgold ON pd.tipo=avgold.tipo
INNER JOIN cant_visitas_2025 cvnew ON cvnew.id=pd.id
WHERE avgold.avg_2024<cvnew.cant_visitas
``` 
