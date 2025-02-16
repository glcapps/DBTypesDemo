# Databases vs. Application Logic: Stop Moving Data and Start Leveraging Your DB
## Start Leveraging Your DB
Many developers still offload data processing to their applications, missing out on the power and efficiency databases provide. Whether it's SQL for complex queries, document databases for flexible data models, or vector databases for specialized searches, modern databases are built to handle computation at scale. By leveraging database capabilities, developers can reduce application overhead, enhance performance, and streamline architecture. With new tools making advanced queries and schema design more accessible, it's now easier than ever to maximize database efficiency. This talk explores how different database types integrate into modern software needs, how AI-assisted tooling enhances ease of use, and why letting the database do the heavy lifting not only leads to more scalable and maintainable applications but also simplifies development in many cases.

[Github Pages Version of DBTypesDemo](https://glcapps.github.io/DBTypesDemo/)

### MacOs
Docker command for Postgres Demo:
docker run --rm --name demo-db -e POSTGRES_PASSWORD=secret -d postgres && \
until docker exec demo-db pg_isready -U postgres; do sleep 1; done && \
docker exec demo-db psql -U postgres -c "
CREATE ROLE anon NOLOGIN;
GRANT USAGE ON SCHEMA public TO anon;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO anon;
CREATE OR REPLACE FUNCTION sql(query TEXT) RETURNS JSONB AS \$\$
DECLARE
    result JSONB;
BEGIN
    EXECUTE format('SELECT jsonb_agg(t) FROM (%s) t', query) INTO result;
    RETURN COALESCE(result, '[]'::JSONB);
END;
\$\$ LANGUAGE plpgsql SECURITY DEFINER;
GRANT EXECUTE ON FUNCTION sql(text) TO anon;
" && \
docker run --rm --name postgrest-demo -p 8088:3000 \
  -e PGRST_DB_URI="postgres://postgres:secret@demo-db/postgres" \
  -e PGRST_DB_ANON_ROLE=anon \
  -e PGRST_DB_SCHEMAS=public \
  -e PGRST_CORS_ALLOWED_ORIGINS="*" \
  --link demo-db postgrest/postgrest

Test shell command: --expect json
  curl -X GET "http://127.0.0.1:8088/" -H "Accept: application/json"

### Windows
Docker command for Postgres Demo (powershell):
docker run --rm --name demo-db -e POSTGRES_PASSWORD=secret -d postgres; `
while (!(docker exec demo-db pg_isready -U postgres)) { Start-Sleep -s 1 }; `
docker exec demo-db psql -U postgres -c "
CREATE ROLE anon NOLOGIN;
GRANT USAGE ON SCHEMA public TO anon;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO anon;
CREATE OR REPLACE FUNCTION sql(query TEXT) RETURNS JSONB AS \$\$
DECLARE result JSONB;
BEGIN
    EXECUTE format('SELECT jsonb_agg(t) FROM (%s) t', query) INTO result;
    RETURN COALESCE(result, '[]'::JSONB);
END;
\$\$ LANGUAGE plpgsql SECURITY DEFINER;
GRANT EXECUTE ON FUNCTION sql(text) TO anon;
"; `
docker run --rm --name postgrest-demo -p 8088:3000 `
  -e PGRST_DB_URI="postgres://postgres:secret@demo-db/postgres" `
  -e PGRST_DB_ANON_ROLE=anon `
  -e PGRST_DB_SCHEMAS=public `
  -e PGRST_CORS_ALLOWED_ORIGINS="*" `
  --link demo-db postgrest/postgrest

Test PowerShell command: --expect json:
Invoke-RestMethod -Uri "http://127.0.0.1:8088/" -Method Get -Headers @{"Accept"="application/json"}