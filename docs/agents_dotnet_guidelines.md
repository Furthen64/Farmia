# AGENTS.md – .NET Development Guardrails

These rules prevent common infrastructure mistakes when working in this repository.

---

# 1. Verify Infrastructure Before Debugging Code

Always confirm the environment is correct before debugging application logic.

Check:
- Configuration loading
- Database migrations
- Correct DbContext usage

Never debug business logic until these are verified.

---

# 2. Configuration Loading

Before testing authentication or config-dependent features:

1. Inspect `Program.cs` to see how configuration is loaded.
2. Ensure `appsettings.json` exists in the runtime directory:

   bin/Debug/net8.0/appsettings.json

3. If multiple projects are involved, explicitly load it:

   builder.Configuration.AddJsonFile("appsettings.json", optional:false, reloadOnChange:true);

4. Add debug logging to confirm values:

   Console.WriteLine($"SharedKey length: {{config["SharedKey"]?.Length}}");

---

# 3. EF Core / DbContext Rules

Before creating or applying migrations:

1. Identify the correct DbContext used by the application.
2. Verify migrations for that context:

   dotnet ef migrations list --context <DbContext>

3. Apply them:

   dotnet ef database update --context <DbContext>

Never assume migrations apply to the correct context.

---

# 4. Database Schema Verification

Before testing features using database tables:

- Confirm required tables exist.
- Verify entity properties match actual columns.
- If schema is wrong, fix migrations.

Do NOT create tables manually with SQL.

---

# 5. Correct Workflow

Always follow this order:

1. Infrastructure check
2. Run migrations
3. Verify configuration values
4. Implement features
5. Test

---

# 6. Rules

- Never debug business logic before verifying infrastructure.
- Always use EF migrations instead of manual SQL.
- Always confirm the correct DbContext.
- Add debug logging when configuration is involved.
- Commit small, incremental changes.
