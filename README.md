
# DevPod + Windows Containers + IIS + .NET Framework (Guia Completo)

Este README explica **passo a passo** como rodar os exemplos fornecidos (HTTP, HTTPS opcional, WebForms, Web Application com MSBuild, solução com múltiplos `.csproj` e API básica) usando **DevPod** em **Windows Containers**.

> **Requisitos rápidos**
>
> - Windows 10/11 **Pro** ou **Enterprise** (ou Windows Server 2019/2022)
> - **Docker Desktop** em modo **Windows Containers** (ícone do Docker → *Switch to Windows Containers…*)
> - **Hyper‑V** habilitado
> - **DevPod** instalado

---

## 1) Preparar o host Windows

1. **Instale o Docker Desktop** e troque para **Windows Containers** (clique no ícone do Docker → *Switch to Windows Containers…*).
2. **Ative o Hyper‑V** (Painel de Controle → Recursos do Windows → Hyper‑V) ou via PowerShell:
   ```powershell
   Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
   ```
3. **Instale o DevPod** (Desktop/CLI) a partir de https://devpod.sh

> Observação: Windows Containers **não** rodam em Linux nativamente; se precisar usar um servidor Linux, utilize uma **VM Windows** para hospedar os containers e o DevPod Agent.

---

## 2) Estrutura dos exemplos

Cada exemplo contém arquivos como:

```
Dockerfile
start.ps1                 # (em alguns templates) script de inicialização / build / deploy / HTTPS
windows-docker-provider.yaml
.devcontainer/
  devcontainer.json       # portas e args do container DevPod
.vscode/
  launch.json             # debug: Attach ao IIS (w3wp)
app/ ou src/              # projetos .csproj e/ou Solution.sln
certs/                    # (somente nos exemplos HTTPS) coloque aqui seu site.pfx
README.md
```

- **Dockerfile**: imagem base Windows SDK 4.8 + IIS.
- **start.ps1** (quando presente): roda `msbuild`, copia artefatos para `C:\inetpub\wwwroot`, faz binding HTTPS (quando configurado) e inicia o IIS com `ServiceMonitor.exe`.
- **windows-docker-provider.yaml**: provider simples para o DevPod usar Docker Windows.
- **devcontainer.json**: publica portas (ex.: `8080:80`, `8443:443`), seleciona `--isolation=hyperv` e injeta VS Code Server.
- **launch.json**: configuração de debug para **Attach ao `w3wp.exe`** (IIS).

---

## 3) Subir o ambiente com DevPod

Na pasta raiz do exemplo:

```powershell
# 1) Adicione o provider (uma vez por máquina)
devpod provider add .\windows-docker-provider.yaml

# 2) Suba o workspace e abra o VSCode conectado ao container
devpod up . --provider windows-docker --ide vscode
```

O DevPod irá:
1. Construir a imagem (Dockerfile)
2. Criar o container Windows
3. Injetar o DevPod Agent e o VS Code Server
4. Abrir o VS Code Desktop **já conectado** ao container

---

## 4) Acessar no navegador

- **HTTP**: `http://localhost:8080`
- **HTTPS (opcional)**: `https://localhost:8443` (somente nos templates com suporte a PFX)

> Para acesso pela LAN, substitua `localhost` por `NOME_DO_PC` ou IP da máquina.

---

## 5) Habilitar HTTPS (apenas templates com HTTPS)

1. Coloque seu certificado **`site.pfx`** em `certs/site.pfx`.
2. Defina a senha do PFX como variável de ambiente no DevPod (Desktop → Workspace → Environment):
   - **`HTTPS_PFX_PASSWORD`** = *sua_senha*
3. Subindo o workspace, o `start.ps1` irá:
   - Importar o PFX para `Cert:\LocalMachine\My`
   - Criar binding HTTPS (`0.0.0.0:443`) no IIS

> **Não** versione o `.pfx` no Git. Use secrets/volumes/cofres em produção.

---

## 6) Compilar sem reiniciar o DevPod (MSBuild dentro do container)

Abra o **Terminal** do VS Code (que já está *dentro* do container) e execute:

- Para solução multi‑projeto:
  ```powershell
  msbuild C:\src\Solution.sln /p:Configuration=Debug
  ```

- Para projeto único:
  ```powershell
  msbuild C:\app\WebApp.csproj /p:Configuration=Debug
  ```

- Publicar/atualizar no IIS (separado do build):
  ```powershell
  Copy-Item C:\src\* C:\inetpub\wwwroot -Recurse -Force
  ```

Assim você **edita → compila → atualiza** sem rebuild da imagem nem restart do DevPod.

---

## 7) Debug no IIS

1. Abra a página no navegador para iniciar o processo **`w3wp.exe`**.
2. No VSCode → **Run** → escolha **Attach IIS (w3wp)**.
3. Coloque breakpoints em `Default.aspx.cs`, controllers da API, etc.

> Dica: se precisar, habilite símbolos/`Just My Code` nas configurações C# do VS Code.

---

## 8) Trabalhando com `.sln` (multi‑projeto)

- Abra a pasta raiz que contém `Solution.sln` no VS Code.
- Use extensões **C# Dev Kit** e **C#** para IntelliSense e navegação.
- O build é feito via **`msbuild` no terminal** (não via *F5*).
- Debug sempre via **Attach ao `w3wp.exe`**.

---

## 9) Dúvidas frequentes

**Posso usar esses exemplos em Linux?**  
Você pode hospedar uma **VM Windows** dentro do Linux e rodar tudo lá. Windows Containers **não** rodam nativamente em Linux.

**Posso compilar automaticamente ao salvar?**  
WebForms recompila dinamicamente `.aspx`, mas para `.csproj`/bibliotecas, rode `msbuild` no terminal e copie arquivos para o IIS (ou automatize no `start.ps1`).

**Como separar o site e a API no IIS?**  
Crie um **aplicativo** `/api` no IIS apontando para a pasta da API e, opcionalmente, um **app pool** dedicado. Posso fornecer um script pronto se desejar.

---

## 10) Troubleshooting rápido

- **`w3wp.exe` não aparece no Attach** → Acesse o site no navegador para inicializar o pool.
- **Portas 8080/8443 ocupadas** → Altere em `.devcontainer/devcontainer.json` → `appPorts`.
- **Erro com PFX** → Verifique a senha (variável `HTTPS_PFX_PASSWORD`) e se o `site.pfx` existe.
- **`msbuild` não encontrado** → Confirme que você está no **terminal do container** (DevPod/VS Code) e que o template usado é o de **SDK + IIS**.

---

## 11) Próximos passos

- Separar **aplicativos e pools** no IIS (WebForms e API isolados)
- Adicionar **SQL Server** (Windows Container) + connection strings
- Integrar **CI/CD** (Azure DevOps/GitHub Actions) para build de imagem
- Colocar **proxy** com HTTPS automático (Traefik/Nginx/Caddy) e expor serviços na LAN/VPN

> Se quiser, peça um template pronto que eu gero com essas opções.

---

### © Feito para ajudar você a desenvolver .NET Framework 4.x em containers Windows com DevPod + VS Code, da forma mais ágil possível.
