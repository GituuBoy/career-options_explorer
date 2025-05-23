2 · Back‑of‑the‑Envelope Design

graph TD
    subgraph "Project Structure"
        Root["/"] --> Client["client/"]
        Root --> Server["server/"]
        Root --> Docs["docs/"]
        Client --> ClientSrc["src/"]
        Client --> ClientPublic["public/"]
        Client --> ClientCfg["config files (tailwind, postcss, vite, tsconfig)"]
        Server --> ServerSrc["src/"]
        Server --> ServerCfg["config files (tsconfig, jest)"]
        Server --> ServerEnv[".env (git‑ignored)"]
    end
    subgraph "Core Tech Stack"
        Client -.-> React["React 18"]
        Client -.-> Vite["Vite"]
        Client -.-> TS_Client["TypeScript"]
        Client -.-> Tailwind["TailwindCSS (iOS Theme)"]
        Server -.-> Node["Node 20 LTS"]
        Server -.-> Express["Express"]
        Server -.-> TS_Server["TypeScript"]
        Server -.-> PostgreSQL["PostgreSQL (via pg driver)"]
    end