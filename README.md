# JSON-with-SQL-Server

--Exemplo 1 : consulta genérica
DECLARE @json AS NVARCHAR(MAX)

SELECT @json = BulkColumn FROM OPENROWSET (BULK 'C:\TEMP\exemplo.json', SINGLE_CLOB) as Arquivo

SELECT F.* FROM OPENJSON(@json) AS F

GO


--Exemplo 2 : nomeando as marcações como nomes de campos
DECLARE @json AS NVARCHAR(MAX)

SELECT @json = BulkColumn 
FROM OPENROWSET (BULK 'C:\TEMP\exemplo.json', SINGLE_CLOB) as Arquivo

SELECT F.*
FROM OPENJSON(@json) 
WITH (
Colecionador   VARCHAR(200)	'$.Colecionador',
Idade          INTEGER      '$.Idade',
Cidade         VARCHAR(200)	'$.Cidade',
Carros         NVARCHAR(MAX) AS JSON 
)AS F


GO

--Exemplo 3 : consultando dados dentro de vetores
DECLARE @json AS NVARCHAR(MAX)

SELECT @json = BulkColumn 
FROM OPENROWSET (BULK 'C:\TEMP\exemplo.json', SINGLE_CLOB) as Arquivo

SELECT Raiz.Colecionador, Raiz.Idade, Raiz.Cidade, Carros.*
FROM OPENJSON(@json) 
WITH (
Colecionador   VARCHAR(200)	'$.Colecionador',
Idade          INTEGER      '$.Idade',
Cidade         VARCHAR(200)	'$.Cidade',
Carros         NVARCHAR(MAX) AS JSON 
)AS Raiz
CROSS APPLY OPENJSON ( Raiz.Carros )
WITH (
Marca                     VARCHAR(50)	'$.Marca',
Cor                       VARCHAR(200)	'$.Cor',
AnoFabricacao             INTEGER       '$.AnoFabricacao',
ValorEstimado             MONEY         '$.ValorEstimado', 
ProprietariosAnteriores   NVARCHAR(MAX) AS JSON
) AS Carros




GO

--Exemplo 4 : gerando métricas
DECLARE @json AS NVARCHAR(MAX)

SELECT @json = BulkColumn 
FROM OPENROWSET (BULK 'C:\TEMP\exemplo.json', SINGLE_CLOB) as Arquivo

;WITH cteMetrica (Colecionador, Idade, Cidade, Marca, AnoFabricacao, ValorEstimado,IdadePropAnt) 
	AS 
	(
	SELECT Raiz.Colecionador, Raiz.Idade, Raiz.Cidade, Carros.Marca,
		Carros.AnoFabricacao, Carros.ValorEstimado, PropAnt.Idade
	FROM OPENJSON(@json) 
	WITH (
	Colecionador   VARCHAR(200)	'$.Colecionador',
	Idade          INTEGER      '$.Idade',
	Cidade         VARCHAR(200)	'$.Cidade',
	Carros         NVARCHAR(MAX) AS JSON 
	)AS Raiz
	CROSS APPLY OPENJSON ( Raiz.Carros )
	WITH (
	Marca                     VARCHAR(50)	'$.Marca',
	Cor                       VARCHAR(200)	'$.Cor',
	AnoFabricacao             INTEGER       '$.AnoFabricacao',
	ValorEstimado             MONEY         '$.ValorEstimado', 
	ProprietariosAnteriores   NVARCHAR(MAX) AS JSON
	) AS Carros
	CROSS APPLY OPENJSON ( Carros.ProprietariosAnteriores)
	WITH (
	Nome                     VARCHAR(200)	'$.Nome',
	Idade                    INTEGER	    '$.Idade',
	Cidade                   VARCHAR(200)   '$.Cidade'
	) AS PropAnt
	WHERE Raiz.Colecionador = 'Zeca Balero'
	),
cteMetPropAnt (Colecionador,CarroMaisAntigo, MediaIdadePropAnt)
	AS (
	SELECT c.Colecionador, MIN(c.AnoFabricacao) AS CarroMaisAntigo, AVG(c.IdadePropAnt) AS MediaIdadePropAnt 
	FROM cteMetrica c
	GROUP BY c.Colecionador
	),
cteMetGeral (Colecionador, Idade, Cidade, Contagem, ValorMedio)
	AS (
	SELECT T.Colecionador, T.Idade, T.Cidade, COUNT(T.Marca) AS Contagem, AVG(T.ValorEstimado) AS ValorMedio
	FROM (
		SELECT DISTINCT c.Colecionador, c.Idade, c.Cidade, c.Marca, c.ValorEstimado
		FROM cteMetrica c
		) T
	GROUP BY T.Colecionador, T.Idade, T.Cidade
	)

SELECT M1.*, M2.*
FROM cteMetGeral M1
	INNER JOIN cteMetPropAnt M2 ON M1.Colecionador = M2.Colecionador

GO
