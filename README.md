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




echo "🚀 Starting PostgreSQL with Apache AGE..."
docker run --rm --name demo-db -e POSTGRES_PASSWORD=secret -d apache/age

echo "⏳ Waiting for PostgreSQL to start..."
until docker exec demo-db pg_isready -U postgres > /dev/null 2>&1; do
  sleep 2
done

echo "🛠️ Installing PostgreSQL development tools and build dependencies..."
docker exec demo-db bash -c "apt-get update && apt-get install -y \
    postgresql-contrib postgresql-server-dev-16 \
    build-essential git cmake libxml2-dev libjson-c-dev flex bison gcc clang"

echo "🛠️ Installing PgVector for PostgreSQL 16 using APT..."
docker exec demo-db bash -c "apt-get update && apt-get install -y postgresql-16-pgvector"

echo "🔌 Enabling PostgreSQL extensions..."
docker exec demo-db psql -U postgres -c "CREATE EXTENSION IF NOT EXISTS age;"
docker exec demo-db psql -U postgres -c "CREATE EXTENSION IF NOT EXISTS vector;"
docker exec demo-db psql -U postgres -c "CREATE EXTENSION IF NOT EXISTS xml2;"
docker exec demo-db psql -U postgres -c "CREATE EXTENSION IF NOT EXISTS cube;"

echo "⚙️ Configuring PostgreSQL roles and functions..."
docker exec demo-db psql -U postgres -c "CREATE ROLE anon NOLOGIN;"
docker exec demo-db psql -U postgres -c "GRANT USAGE ON SCHEMA public TO anon;"
docker exec demo-db psql -U postgres -c "GRANT SELECT ON ALL TABLES IN SCHEMA public TO anon;"

echo "🛠️ Creating PostgREST-compatible SQL function..."
docker exec demo-db psql -U postgres -c "
CREATE OR REPLACE FUNCTION public.sql(query TEXT) RETURNS JSONB AS \$\$
DECLARE
    stmt TEXT;
    result JSONB := '[]'::JSONB;
    query_array TEXT[];
    temp_result JSONB;
    i INT;
BEGIN
    -- Split queries on semicolon (;)
    query_array := string_to_array(query, ';');

    -- Loop through each statement and execute it
    FOR i IN 1..array_length(query_array, 1) LOOP
        stmt := trim(query_array[i]);

        -- Skip empty statements
        IF stmt <> '' THEN
            -- If it's a SELECT, return results as JSON
            IF lower(stmt) LIKE 'select%' THEN
                EXECUTE format('SELECT jsonb_agg(t) FROM (%s) t', stmt) INTO temp_result;
                -- Append results to the final JSON array
                result := result || COALESCE(temp_result, '[]'::JSONB);
            ELSE
                -- If it's DDL/DML (CREATE, INSERT, DROP, etc.), execute but don’t return results
                EXECUTE stmt;
            END IF;
        END IF;
    END LOOP;

    RETURN result;
END;
\$\$ LANGUAGE plpgsql SECURITY DEFINER;
GRANT EXECUTE ON FUNCTION public.sql(TEXT) TO anon;
"

echo "✅ Installed PostgreSQL Extensions:"
docker exec demo-db psql -U postgres -c "\dx"

echo "🚀 Restarting PostgREST to refresh the schema cache..."
docker stop postgrest-demo >/dev/null 2>&1 || true
docker run --rm --name postgrest-demo -p 8088:3000 \
  -e PGRST_DB_URI="postgres://postgres:secret@demo-db/postgres" \
  -e PGRST_DB_ANON_ROLE=anon \
  -e PGRST_DB_SCHEMAS=public,ag_catalog \
  -e PGRST_DB_SCHEMAS=public \
  -e PGRST_DB_SCHEMA_CACHE_TTL=0 \
  -e PGRST_CORS_ALLOWED_ORIGINS="*" \
  --link demo-db postgrest/postgrest