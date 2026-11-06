SELECT DISTINCT
       OBJECT_SCHEMA_NAME(d.referencing_id) + N'.' + OBJECT_NAME(d.referencing_id) AS ReferencingObject,
       o.type_desc AS ObjectType,
       OBJECT_NAME(o.parent_object_id) AS ParentObject
FROM sys.sql_expression_dependencies d
JOIN sys.objects o ON o.object_id = d.referencing_id
WHERE d.referenced_id IN (SELECT object_id FROM sys.tables WHERE name LIKE 'PlanOfWork%' OR name = 'SourceOfFundingAllocation')
  AND d.is_schema_bound_reference = 1;