# MySql_PostgreSQL

Se usaron sentencias create table y alter table para crear un pequeño esquema donde se muestra el funcionamiento de
algunas restricciones de forma declarativa (restricciones de integridad, restricciones de integridad referencial y
reglas de negocio). Algunas reglas de negocio se hicieron de forma procedural (funciones y trigger). También se usan
vistas, algunas actualizables y otras no.

--Creacion de tabla:

CREATE TABLE `reciclados` (

  `id` int NOT NULL AUTO_INCREMENT,
  `bottles` int NULL,
  `tetrabriks` int NULL,
  `glass` int NULL,
  `paperboard` int NULL,
  `cans` int NULL,
  `date` date NULL,
  `id_r` int NOT NULL,
  PRIMARY KEY (`id`),
  FOREIGN KEY (`id_r`) REFERENCES `usuario` (`id`)
  
);

CREATE TABLE `usuario` (

  `id` int NOT NULL AUTO_INCREMENT,
  `firstname` varchar(20) NULL,
  `lastname` varchar(20)  NULL,
  `username` varchar(20)  NOT NULL,
  `address` varchar(20)  NULL,
  `mail` varchar(20)  NULL,
  PRIMARY KEY (`id`)
  
);

---------------------------------------------------------------------------------

--Alteracion de tabla, PK, FK, ADDCOLUMN

ALTER TABLE Trabaja
ADD CONSTRAINT PK_Trabaja
PRIMARY KEY (id_ingeniero,id_sector,nro_proyecto);

ALTER TABLE mesa
ADD CONSTRAINT FK_RESTAURANT_MESA
FOREIGN KEY (codigo)
REFERENCES mesa (codigo);

ALTER TABLE teatro
ADD COLUMN cantidad integer default 0 not null;

---------------------------------------------------------------------------------

--Declaracion de FK para alquiler_deposito y uso de acciones referenciales y tipos de match (Posibles valores nulos para FK).

CONSTRAINT FK_MESA_REST FOREIGN KEY (codigo) REFERENCES restaurant (codigo)
ON UPDATE CASCADE
ON DELETE CASCADE
Match Partial -- Simple-Full
);

---------------------------------------------------------------------------------

--Tipos y Dominios

CREATE TYPE tipo_direccion AS (
	calle varchar(60),
	altura decimal(5,0) 
);

direccion tipo_direccion not null,

CREATE DOMAIN sueldo_valido AS				
	decimal(7,2) not null
	CHECK (value between 0 and 5);
	CHECK (value LIKE 'ARG' or
	       value LIKE 'CHI');
         
sueldo sueldo_valido not null,

---------------------------------------------------------------------------------

--Restricciones declarativas (dominio/atributo - tupla - tabla - general)

Tablas afectadas:
Atributos afectados:
Tipo de Restriccion: dominio/atributo - tupla - tabla - general

-- Tablas: alquiler
-- Atributos: id_alquiler, id_cliente, fecha_hasta
-- Tipo de Restriccion: tabla porque invlucra a un conjunto de tuplas.

ALTER TABLE alquiler
ADD CONSTRAINT chk_max_5
CHECK ( NOT EXISTS (select 1
                    from alquiler
                    where fecha_hasta is Null
                    group by (id_cliente)
                    having count (*) > 5) );

-- Tablas: tipo_espacio
-- Atributos: volumen, costo
-- Tipo de Restriccion: tupla ya que invucra a un cojunto de atributos.

ALTER TABLE tipo_espacio
ADD CONSTRAINT chk_b
CHECK ( (volum_max < 10 and costo_diario < 100) OR (volum_max > 10) );

-- Tablas: cliente, alquiler, alquiler_deposito, deposito
-- Atributos: id_cliente, fecha_alta, ubicacion
-- Tipo de Restriccion: assertion, necesitamos mas de una tabla para satisfacer esta consulta.

CREATE ASSERTION ejercicio_c
CHECK ( NOT EXISTS (select 1
                    from cliente c join alquiler a on (c.id_cliente = a.id_cliente) join alquiler_deposito ad on (a.id_alquiler = ad.id_alquiler)
                        join deposito d on (ad.nro_dep = d.nro_dep and ad.id_espacio = d.id_espacio) 
                    where ( (current_date - fecha_alta) < 365 ) and (ubicacion like 'zona preferencial') ) ) );

---------------------------------------------------------------------------------

--Uso de Triggers 

CREATE ASSERTION ejercicio_d
CHECK ( NOT EXISTS (select 1
                    from alquiler_deposito ad join alquiler a on (ad.id_alquiler = a.id_alquiler) join tipo_espacio te on
                    (ad.id_espacio = te.id_espacio) 
                    where te.costo_diario > (a.importe_dia * 1.2) ) );
                    
Ya que la implementacion de los Assertion no los permite ningun DBMS se programa un Trigger para resolver esta regla de negocio.

-------------------------------------------------

-- Evento critico: Insert y Update en la tabla alquiler_deposito.
-- t de activacion: After.
-- Granularidad: for each row (una vez por cada fila afectada).

create or replace function fn_b() returns trigger as $$

begin
	
	if ( NOT EXISTS (select 1
                    from alquiler_deposito ad join alquiler a on (ad.id_alquiler = a.id_alquiler) join tipo_espacio te on
                    (ad.id_espacio = te.id_espacio) 
                    where te.costo_diario > (a.importe_dia * 1.2) ) ) then

        return new;

	end if;
	
    RAISE EXCEPTION 'No se puede insertar o actualizar en dicha tabla. Verifique el importe_dia y costo_diario';

end $$
language 'plpgsql';

create trigger tg_b After Insert or Update on alquiler_deposito for each row execute procedure fn_b();


-- Acitvacion del trigger

--Suponiendo que el id_alquiler = 6 tiene importe_dia superior a costo_diario de id_espacio = 55 no lo va a permitir agregar.
INSERT INTO alquiler_deposito (id_alquiler,nro_dep,id_espacio,estado) values (6, 1, 55, True);

-- Mismo ejemplo de arriba, quiero actualizar id_espacio = 55 con id_alquiler = 6 y nro_dep = 1.
update alquiler set id_espacio = 6 where id_alquiler = 6 and nro_dep = 1;

---------------------------------------------------------------------------------

--Vistas

CREATE VIEW Vista_azul AS
select *
from vehiculo
where tipo = 'auto' and color = 'azul';

 Uso de WITH [LOCAL-CASCADE] CHECK OPTION; solucion a la migracion de tuplas. Solo en vistas automaticamente actualizables.

update Vista_azul set color = 'rojo'; 

Si tiene el WITH y se quiere hacer update Vista_azul set color = 'rojo'; no lo va a permitir, ya que ante una modificacion primero tiene que hacer TRUE la condicion del where de la vista.
