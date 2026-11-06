INSERT #Problems
SELECT DISTINCT N'Schema-bound object blocks rename: '
     + OBJECT_SCHEMA_NAME(d.referencing_id) + N'.' + OBJECT_NAME(d.referencing_id)
FROM sys.sql_expression_dependencies d
JOIN #Scope s ON s.object_id = d.referenced_id
JOIN sys.objects o ON o.object_id = d.referencing_id
WHERE d.is_schema_bound_reference = 1
  AND NOT (o.type IN ('C', 'D') AND o.parent_object_id = d.referenced_id);