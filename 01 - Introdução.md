## 🚀 Sobre a Linguagem Go (Golang)

- **Linguagem Compilada:** O código-fonte é traduzido diretamente para código de máquina antes da execução, o que geralmente resulta em melhor performance.
    
- **Fortemente Tipada:** Toda variável tem um tipo de dado específico (como `int`, `string`, `bool`) e esse tipo **não pode ser alterado** após a variável ser declarada.
    
- **Origem:** É uma linguagem derivada de C/C++, buscando modernizar e simplificar muitos de seus conceitos.
    

---

## 🖥️ Configurando o Ambiente no Windows

1. Acesse o site oficial para fazer o download: `golang.org`
    
2. Baixe e execute o arquivo de instalação (geralmente um `.msi`).
    
3. Siga os passos do assistente de instalação (geralmente, as opções padrão são suficientes).
    
4. O instalador padrão geralmente adiciona o Go ao **PATH** do Windows automaticamente. Esta etapa é crucial para que você possa executar comandos `go` de qualquer terminal.
    

---

## ✅ Verificando a Instalação

Depois de instalar, abra um novo terminal (Prompt de Comando ou PowerShell) e execute os seguintes comandos:

### 1. Verificar a Versão

Este é o teste mais simples para confirmar que o Go foi instalado e reconhecido pelo sistema.

- Execute o comando:
    
    
    ```bash
    go version
    ```
    
- **Resultado esperado:** Você verá a versão do Go que acabou de instalar (ex: `go version go1.21.0 windows/amd64`).
    

### 2. Verificar as Variáveis de Ambiente

Isso mostra todas as configurações que o Go está usando.

- Execute o comando:
   
```bash
   go env
```
- **Resultado esperado:** O comando listará diversas variáveis. Se o comando for reconhecido e não der erro, a instalação está correta.
    
- **Variável importante:** Preste atenção na variável `GOPATH`. Ela define o caminho da sua "área de trabalho" (workspace), onde o Go salva pacotes baixados e os binários dos seus projetos.