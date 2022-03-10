# FNZ

missing  header
/***
<StoredProcedure>
    <Description>Run Conversion Disaggregation.</Description>
	<Parameters>
        <Parameter Name="@conversion_id">
            <Description></Description>
        </Parameter>
    </Parameters>
</StoredProcedure>
***/
CREATE PROCEDURE [FundConversionETL].[spRunConversionDisaggregation]
(
    @conversion_id INT
)   
AS
thanks

/** Testing
DECLARE @conversion_id INT
SET @conversion_id = 23284
**/

DECLARE @InstructedQuantity DECIMAL(18, 8)
DECLARE @ConversionQuantity DECIMAL(18, 8) 
DECLARE @InstrumentDp TINYINT 

CREATE TABLE #Holdings
(
    [SEClientAccountId]       INT            NOT NULL,
    [InstrumentId]            INT            NOT NULL,
    [CustodianAccountId]      VARCHAR(30) COLLATE SQL_LATIN1_GENERAL_CP1_CI_AS NOT NULL, --are accountIds not len 20? ok
    [Quantity]                DECIMAL(18, 8) NOT NULL,
    [Group1Quantity]          DECIMAL(18, 8) NULL,
    [Group2Quantity]          DECIMAL(18, 8) NULL,
	[OldGroup1Quantity]       DECIMAL(18, 8) NULL,
    [OldGroup2Quantity]       DECIMAL(18, 8) NULL,
    [ConvertedQuantity]       DECIMAL(18, 8) NULL,
    [ConvertedGroup1Quantity] DECIMAL(18, 8) NULL,
    [ConvertedGroup2Quantity] DECIMAL(18, 8) NULL
)

CREATE TABLE #GroupHoldings
(
    [SEClientAccountId] INT            NOT NULL,
    [InstrumentId]      INT            NOT NULL,
    [Group1]            DECIMAL(18, 8) NOT NULL,
    [Group2]            DECIMAL(18, 8) NOT NULL
)

INSERT INTO #Holdings
    ([SEClientAccountId],
     [InstrumentId],
     [CustodianAccountId],
     [Quantity])
SELECT 
    [SEClientAccountId],
    [InstrumentId],
    [CustodianAccountId],
    [Quantity]
FROM
    [Platform].[FundConversionLogging].[RingfencedHoldings]
WHERE
    [ConversionId] = @conversion_id
	AND Quantity > 0

CREATE NONCLUSTERED INDEX idx ON #Holdings (SEClientAccountId, InstrumentId) INCLUDE (CustodianAccountId)

SELECT
    @InstructedQuantity = InstructedQuantity,
    @ConversionQuantity = ConversionQuantity,
    @InstrumentDp = TargetInstrumentUnitDp
FROM
    [FundConversionETL].[InstrumentConversions]
WHERE
    ConversionQuantity IS NOT NULL

-- Group holdings
IF EXISTS (    
    SELECT 
        1 
    FROM 
        [Platform].[FundConversion].[Config] 
    WHERE 
        [ConfigVariable] = 'UseOldGroupSystem' 
        AND [ConfigValue] = '1'
)
BEGIN
    INSERT INTO #GroupHoldings (SEClientAccountId, InstrumentId, Group1, Group2)
    SELECT
        h.SEClientAccountId,
        h.InstrumentId,
        stt.Group1,
        stt.Group2
    FROM
        #Holdings h
        INNER JOIN [Platform].MarketData.Instruments i ON h.InstrumentId = i.Id
        INNER JOIN ClientAccount.dbo.vwScripTransactionsTotals_OldGroupSystem_Indexed stt ON h.SEClientAccountId = stt.AccountId
            AND i.[Security] = stt.InstrumentCode
        INNER JOIN [Platform].Transactions.Locations l ON l.Id = stt.[LocationId]
END
ELSE
BEGIN
    INSERT INTO #GroupHoldings (SEClientAccountId, InstrumentId, Group1, Group2)
    SELECT
        h.SEClientAccountId,
        h.InstrumentId,
        stt.Group1,
        stt.Group2
    FROM
        #Holdings h
        INNER JOIN [Platform].MarketData.Instruments i ON h.InstrumentId = i.Id
        INNER JOIN ClientAccount.dbo.vwScripTransactionsTotals_Indexed stt ON h.SEClientAccountId = stt.AccountId
            AND i.[Security] = stt.InstrumentCode
        INNER JOIN [Platform].Transactions.Locations l ON l.Id = stt.[LocationId]
END

CREATE NONCLUSTERED INDEX idx_SEClientAccountIdInstrumentId ON #GroupHoldings (SEClientAccountId, InstrumentId) INCLUDE (Group1, Group2)

-- Set the groups (while pulling override values for single account conversions)
UPDATE
    h
SET
    h.OldGroup1Quantity = COALESCE(rgh.Group1Quantity, gh.Group1),
    h.OldGroup2Quantity = COALESCE(rgh.Group2Quantity, gh.Group2)
FROM
    #Holdings h
    INNER JOIN #GroupHoldings gh ON h.SEClientAccountId = gh.SEClientAccountId
        AND h.InstrumentId = gh.InstrumentId
    LEFT JOIN [Platform].[FundConversionLogging].[RingfencedGroupHoldings] rgh ON h.SEClientAccountId = rgh.SEClientAccountId
        AND h.InstrumentId = rgh.InstrumentId

DROP TABLE #GroupHoldings

--Remove anything with zero group holdings (shouldn't happen, but just protecting against divide by zero)
DELETE FROM 
	#Holdings
WHERE
	(OldGroup1Quantity + OldGroup2Quantity) = 0

--With DIMs trading accounts, ringfenced quantity likely won't equal full quantity of holdings in that instrument
--So make sure the group quantities are inline with the total ringfenced quantity, instead of total current holdings quantity
UPDATE
	#Holdings
SET
	Group1Quantity = ROUND((OldGroup1Quantity / (OldGroup1Quantity + OldGroup2Quantity)) * Quantity, @InstrumentDp, 1),
	Group2Quantity = ROUND((OldGroup2Quantity / (OldGroup1Quantity + OldGroup2Quantity)) * Quantity, @InstrumentDp, 1)

--Similar to below, attribute any fractional rounding to group 1
UPDATE
    #Holdings
SET
    Group1Quantity = Group1Quantity + (Quantity - (Group1Quantity + Group2Quantity))
WHERE
    Quantity <> Group1Quantity + Group2Quantity 
    AND Quantity = (Group1Quantity + (Quantity - (Group1Quantity + Group2Quantity))) + Group2Quantity

-- Pass 1: Quantity disaggregation
UPDATE
    #Holdings
SET
    ConvertedQuantity = ROUND((Quantity / @InstructedQuantity) * @ConversionQuantity, @InstrumentDp, 1)

-- Pass 2: Group 1/2 disaggregation
UPDATE
    #Holdings
SET
    ConvertedGroup1Quantity = ROUND((Group1Quantity / Quantity) * ConvertedQuantity, @InstrumentDp, 1),
    ConvertedGroup2Quantity = ROUND((Group2Quantity / Quantity) * ConvertedQuantity, @InstrumentDp, 1)
not tha I saw
 
-- Pass 3: Attribute any fractional rounding to group 1
UPDATE
    #Holdings
SET
    ConvertedGroup1Quantity = ConvertedGroup1Quantity + (ConvertedQuantity - (ConvertedGroup1Quantity + ConvertedGroup2Quantity))
WHERE
    ConvertedQuantity <> ConvertedGroup1Quantity + ConvertedGroup2Quantity 
    AND ConvertedQuantity = (ConvertedGroup1Quantity + (ConvertedQuantity - (ConvertedGroup1Quantity + ConvertedGroup2Quantity))) + ConvertedGroup2Quantity
   
INSERT INTO [FundConversionETL].[ConvertedHoldings]
    ([SEClientAccountId],
     [InstrumentId],
     [CustodianAccountId],
     [Quantity],
     [Group1Quantity],
     [Group2Quantity],
     [ConvertedQuantity],
     [ConvertedGroup1Quantity],
     [ConvertedGroup2Quantity])
SELECT
    [SEClientAccountId],
    [InstrumentId],
    [CustodianAccountId],
    [Quantity],
    [Group1Quantity],
    [Group2Quantity],
    [ConvertedQuantity],
    [ConvertedGroup1Quantity],
    [ConvertedGroup2Quantity]
FROM
    #Holdings
	
DROP TABLE #Holdings
