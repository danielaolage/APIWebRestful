# APIWebRestful
APIWebRestful em .NET 5.0 com injeção de dependência e dois métodos HTTP: GET e POST

















USE [LFC_CONTROLE_PERDAS]
GO
/****** Object:  Schema [tmp]    Script Date: 8/10/2023 12:35:50 PM ******/
CREATE SCHEMA [tmp]
GO
/****** Object:  StoredProcedure [dbo].[ADICIONAR_SUB_AREAS_EM_GEPOR]    Script Date: 8/10/2023 12:35:50 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO


CREATE PROCEDURE [dbo].[ADICIONAR_SUB_AREAS_EM_GEPOR]
AS
BEGIN
	DECLARE @idGepor int
	DECLARE @nomeArea varchar(MAX)
	DECLARE @nomeSubArea varchar(MAX)
	DECLARE cursorAreasGepor CURSOR
	STATIC FOR
		SELECT a.ID, a.NOME
		FROM AREA_APONTAMENTO_EVENTO as a
		WHERE a.NOME like 'GEPOR'

	OPEN cursorAreasGepor
		IF @@CURSOR_ROWS > 0
		BEGIN
			FETCH NEXT FROM cursorAreasGepor INTO @idGepor, @nomeArea
			WHILE @@Fetch_status = 0
			BEGIN
				PRINT 'Area GEPOR ID: ' + CAST(@idGepor as varchar(MAX))
				DECLARE cursorSubAreasNaoCadastradas CURSOR FOR
					(SELECT DISTINCT sb.NOME
					FROM AREA_APONTAMENTO_EVENTO as a
					LEFT JOIN SUB_AREA_APONTAMENTO_EVENTO as sb
					ON a.ID = sb.AREA_APONTAMENTO_EVENTO_ID
					WHERE a.NOME like 'Porto' and 	sb.NOME not in (SELECT sb.NOME FROM
													SUB_AREA_APONTAMENTO_EVENTO as sb
													WHERE sb.AREA_APONTAMENTO_EVENTO_ID = @idGepor ))
					OPEN cursorSubAreasNaoCadastradas
						IF @@CURSOR_ROWS > 0
						BEGIN
							FETCH NEXT FROM cursorSubAreasNaoCadastradas INTO @nomeSubArea
							WHILE @@Fetch_status = 0
							BEGIN
								PRINT 'SubArea: ' + @nomeSubArea
									INSERT INTO [dbo].[SUB_AREA_APONTAMENTO_EVENTO]
											   ([AREA_APONTAMENTO_EVENTO_ID]
											   ,[NOME])
										 VALUES
											   (@idGepor 
											   ,@nomeSubArea)
								FETCH NEXT FROM cursorSubAreasNaoCadastradas INTO @nomeSubArea
							END
						END
					CLOSE cursorSubAreasNaoCadastradas;
					DEALLOCATE cursorSubAreasNaoCadastradas;
				FETCH NEXT FROM cursorAreasGepor INTO @idGepor, @nomeArea
			END
		END
	CLOSE cursorAreasGepor;
	DEALLOCATE cursorAreasGepor;
END
GO
/****** Object:  StoredProcedure [dbo].[insereArea]    Script Date: 8/10/2023 12:35:50 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[insereArea]
	@NomeArea varchar(200),
	@NomeSubAreas varchar(400)
AS
BEGIN
	DECLARE @idlinha int
	DECLARE @idarea int
	DECLARE @linha varchar(MAX)
	DECLARE @subArea varchar(MAX)
	
	DECLARE cursorlinhas CURSOR
	STATIC FOR
		SELECT l.ID, l.LINHA
		FROM [LINHA_APONTAMENTO_EVENTO] as l
		WHERE l.DELETADO is null or l.DELETADO = 0

	SET XACT_ABORT ON
	
	BEGIN TRY
		BEGIN TRAN 
			OPEN cursorlinhas
				IF @@CURSOR_ROWS > 0
				BEGIN
					FETCH NEXT FROM cursorlinhas INTO @idlinha, @linha
					WHILE @@Fetch_status = 0
					BEGIN
						INSERT INTO [dbo].[AREA_APONTAMENTO_EVENTO]
								   ([LINHA_APONTAMENTO_EVENTO_ID]
								   ,[NOME]
								   ,[DELETADO])
							VALUES
								   (@idlinha
								   ,@NomeArea
								   ,0)
						SELECT @idarea = SCOPE_IDENTITY()
							DECLARE cursorSubAreas CURSOR
							STATIC FOR
								SELECT Item FROM [dbo].[SplitInts] (
								   @NomeSubAreas
								  ,';')
							OPEN cursorSubAreas
								IF @@CURSOR_ROWS > 0
								BEGIN
									FETCH NEXT FROM cursorSubAreas INTO @subArea
									WHILE @@Fetch_status = 0
									BEGIN
										PRINT 'Linha: ' + @linha + ' ID: ' + CAST(@idlinha as varchar(MAX)) + ' Area: ' + @NomeArea + ' ID: ' + CAST(@idarea as varchar(MAX)) + ' Sub-area: ' + @subArea;
										INSERT INTO [dbo].[SUB_AREA_APONTAMENTO_EVENTO]
										           ([AREA_APONTAMENTO_EVENTO_ID]
										           ,[NOME]
										           ,[DELETADO])
										     VALUES
										           (@idarea
										           ,@subArea
										           ,0)
										FETCH NEXT FROM cursorSubAreas INTO @subArea
									END
								END
							CLOSE cursorSubAreas;
							DEALLOCATE cursorSubAreas;
						FETCH NEXT FROM cursorlinhas INTO @idlinha, @linha
					END
				END
			CLOSE cursorlinhas;
			DEALLOCATE cursorlinhas;
		--THROW 51000, 'The record does not exist.', 1;  
		COMMIT
	END TRY
	BEGIN CATCH
		PRINT 'Error ' + CONVERT(varchar(50), ERROR_NUMBER()) +
			  ', Severity ' + CONVERT(varchar(5), ERROR_SEVERITY()) +
			  ', State ' + CONVERT(varchar(5), ERROR_STATE()) + 
			  ', Procedure ' + ISNULL(ERROR_PROCEDURE(), '-') + 
			  ', Line ' + CONVERT(varchar(5), ERROR_LINE());
		PRINT ERROR_MESSAGE();
		ROLLBACK     
		RETURN -1 
	END CATCH
	
	SET XACT_ABORT OFF
END
GO
/****** Object:  StoredProcedure [dbo].[removerDuplicadosEquipamento]    Script Date: 8/10/2023 12:35:50 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO


CREATE PROCEDURE [dbo].[removerDuplicadosEquipamento]
AS
BEGIN
	-- SET NOCOUNT ON added to prevent extra result sets from
	-- interfering with SELECT statements.
	SET NOCOUNT OFF;
	
	DELETE FROM [LFC_CONTROLE_PERDAS].[dbo].[EQUIPAMENTO_APONTAMENTO_EVENTO]
	WHERE [LFC_CONTROLE_PERDAS].[dbo].[EQUIPAMENTO_APONTAMENTO_EVENTO].ID IN (
		SELECT equipamento.ID 
		FROM [LFC_CONTROLE_PERDAS].[dbo].[EQUIPAMENTO_APONTAMENTO_EVENTO] as equipamento
		LEFT OUTER JOIN (
		   SELECT MIN(ID) as ID, NOME
		   FROM [LFC_CONTROLE_PERDAS].[dbo].[EQUIPAMENTO_APONTAMENTO_EVENTO] 
		   GROUP BY NOME
		) as KeepRows ON
		   equipamento.ID = KeepRows.ID
		WHERE
		   KeepRows.ID IS NULL AND equipamento.ID NOT IN (
			SELECT equipamento.ID
			FROM [LFC_CONTROLE_PERDAS].[dbo].[EQUIPAMENTO_APONTAMENTO_EVENTO] as equipamento 
			INNER JOIN [LFC_CONTROLE_PERDAS].[dbo].[APONTAMENTO_EVENTO]  as apontamento
			on equipamento.ID = apontamento.EQUIPAMENTO_APONTAMENTO_EVENTO_ID
		   )
	);
	
	DECLARE @count int
	DECLARE @Id int
	DECLARE @nome varchar(MAX)
	DECLARE @IdDuplicado int
	DECLARE @nomeDuplicado varchar(MAX)
	DECLARE @IdApontamento int
	DECLARE @inicio datetime
	DECLARE listaEquipamentoRepetidoComApontamento CURSOR
		STATIC FOR
			SELECT MIN(ID) as ID, NOME
			FROM [LFC_CONTROLE_PERDAS].[dbo].[EQUIPAMENTO_APONTAMENTO_EVENTO] 
			GROUP BY NOME
			HAVING COUNT(*) > 1 
			ORDER BY COUNT(*) DESC;

	  OPEN listaEquipamentoRepetidoComApontamento
	  IF @@CURSOR_ROWS > 0
	   BEGIN
	   FETCH NEXT FROM listaEquipamentoRepetidoComApontamento INTO @Id, @nome 
	   WHILE @@Fetch_status = 0
	   BEGIN
		PRINT 'ID: '+ convert(varchar(20), @Id) + ', Nome do Equipamento : '+@nome
		
		DECLARE cursorEquipamentoNome CURSOR FOR
		SELECT ID, NOME 
		FROM [LFC_CONTROLE_PERDAS].[dbo].[EQUIPAMENTO_APONTAMENTO_EVENTO] 
		Where NOME like @nome and ID != @Id;
			OPEN cursorEquipamentoNome;
			FETCH NEXT FROM cursorEquipamentoNome INTO @IdDuplicado, @nomeDuplicado ;
			WHILE @@FETCH_STATUS = 0
			BEGIN
				PRINT ' - Duplicado ID: '+ convert(varchar(20), @IdDuplicado) + ', Nome do Equipamento : '+@nomeDuplicado
					DECLARE cursorApontamento CURSOR FOR
						SELECT ID, INICIO 
						FROM [LFC_CONTROLE_PERDAS].[dbo].[APONTAMENTO_EVENTO] 
						Where EQUIPAMENTO_APONTAMENTO_EVENTO_ID=@IdDuplicado;
					OPEN cursorApontamento;
					FETCH NEXT FROM cursorApontamento INTO @IdApontamento, @inicio;
					WHILE @@FETCH_STATUS = 0
					BEGIN
						PRINT ' | - Apontamento: ' + Cast(@IdApontamento as Varchar) + ' Inicio: ' + Cast(@inicio as Varchar) ;
						
						UPDATE [LFC_CONTROLE_PERDAS].[dbo].[APONTAMENTO_EVENTO] 
						SET EQUIPAMENTO_APONTAMENTO_EVENTO_ID = @Id
						WHERE ID = @IdApontamento
						SET @count = @count + 1;

						FETCH NEXT FROM cursorApontamento INTO @IdApontamento, @inicio;
					END;
					CLOSE cursorApontamento;
					DEALLOCATE cursorApontamento;
				
				FETCH NEXT FROM cursorEquipamentoNome INTO @IdDuplicado, @nomeDuplicado ;
			END;
			CLOSE cursorEquipamentoNome;
			DEALLOCATE cursorEquipamentoNome;

		FETCH NEXT FROM listaEquipamentoRepetidoComApontamento INTO @Id, @nome
	   END
	  END
	  CLOSE listaEquipamentoRepetidoComApontamento;
	  DEALLOCATE listaEquipamentoRepetidoComApontamento;

	DELETE FROM [LFC_CONTROLE_PERDAS].[dbo].[EQUIPAMENTO_APONTAMENTO_EVENTO]
	WHERE [LFC_CONTROLE_PERDAS].[dbo].[EQUIPAMENTO_APONTAMENTO_EVENTO].ID IN (
		SELECT equipamento.ID 
		FROM [LFC_CONTROLE_PERDAS].[dbo].[EQUIPAMENTO_APONTAMENTO_EVENTO] as equipamento
		LEFT OUTER JOIN (
		   SELECT MIN(ID) as ID, NOME
		   FROM [LFC_CONTROLE_PERDAS].[dbo].[EQUIPAMENTO_APONTAMENTO_EVENTO] 
		   GROUP BY NOME
		) as KeepRows ON
		   equipamento.ID = KeepRows.ID
		WHERE
		   KeepRows.ID IS NULL AND equipamento.ID NOT IN (
			SELECT equipamento.ID
			FROM [LFC_CONTROLE_PERDAS].[dbo].[EQUIPAMENTO_APONTAMENTO_EVENTO] as equipamento 
			INNER JOIN [LFC_CONTROLE_PERDAS].[dbo].[APONTAMENTO_EVENTO]  as apontamento
			on equipamento.ID = apontamento.EQUIPAMENTO_APONTAMENTO_EVENTO_ID
		   )
	);
END
GO
