### 1. Visão Geral

No ecossistema Go, a linguagem não possui uma palavra-chave nativa `yield` para criar geradores preguiçosos (_lazy evaluation_) como encontrado em Python ou JavaScript. Em vez disso, o **Padrão Generator** é implementado idiomaticamente combinando **Goroutines** e **Channels**. Uma função atua como fábrica: ela inicializa um canal, despacha uma Goroutine em _background_ para popular esse canal produzindo dados assincronamente, e imediatamente retorna o canal (tipicamente _read-only_) para o chamador. O problema central que este padrão resolve é o desacoplamento entre produtor e consumidor e a eficiência de memória: permite a iteração sobre fluxos de dados infinitos, paginação massiva de banco de dados ou pipelines de processamento pesado sem a necessidade de alocar todo o _dataset_ previamente na memória (Slice).

### 2. Organização por Tópicos

O domínio profundo do Padrão Generator exige o controle das seguintes mecânicas:

- **Fábrica de Canais (O Gerador Básico):** A arquitetura base de instanciar um canal, iniciar a Goroutine e retornar a via de leitura (`<-chan T`).
    
- **Sinalização de Fim (Graceful Close):** A responsabilidade estrita do produtor (a Goroutine geradora) em fechar o canal via `defer close(ch)` para avisar o consumidor que não há mais dados.
    
- **Prevenção de Goroutine Leaks (Padrão Sênior):** A obrigatoriedade absoluta de injetar um mecanismo de controle (como `context.Context` ou um canal `done`) para abortar a Goroutine geradora caso o consumidor abandone a leitura prematuramente.
    

### 3. Visualização do Fluxo (Mermaid)

Snippet de código

```
sequenceDiagram
    participant Caller as Escopo Consumidor (Main)
    participant Gen as Função Geradora
    participant Chan as Canal (<-chan)
    participant Worker as Goroutine (Produtor)

    Caller->>Gen: Invoca StreamData(ctx)
    Gen->>Gen: Inicializa make(chan Data)
    Gen->>Worker: Despacha via 'go func()'
    Gen-->>Caller: Retorna Canal de Leitura
    
    Note right of Worker: Execução Concorrente Inicia
    
    loop Enquanto houver demanda/dados
        Worker->>Chan: Envia dado (ch <- data)
        Chan-->>Caller: Extrai dado (<-ch)
    end
    
    alt Caller interrompe leitura (Cancel)
        Caller->>Worker: Dispara ctx.Done()
        Worker->>Chan: close(ch)
        Worker->>Worker: Retorna e morre (Evita Leak)
    else Fim natural dos dados
        Worker->>Chan: close(ch)
        Worker->>Worker: Retorna e morre
    end
```

**Implementação Passo a Passo (Diagrama):**

- **Invocação não-bloqueante:** Quando `Main` chama a fábrica geradora, a função retorna quase instantaneamente, devolvendo o controle e a referência do canal.
    
- **O Canal como Buffer:** A comunicação ocorre. O produtor `Worker` envia um dado. Se for um canal não-bufferizado, o produtor dorme até o `Caller` extrair esse dado.
    
- **O Risco de Vazamento (Goroutine Leak):** Se o `Caller` decidir que encontrou o dado que queria e parar de ler o canal no meio do loop, o `Worker` ficará bloqueado eternamente na linha `ch <- data` (tentando enviar para um canal cheio/sem leitor). A injeção de contexto (`ctx.Done()`) é a via de escape que permite ao _Scheduler_ do Go matar a Goroutine órfã.
    

### 4 e 5. Exemplos de Código (Idiomático) e Implementação Passo a Passo

#### Tópico A: O Gerador Básico (Fluxo Finito)

Go

```
package concurrency

import "fmt"

// GenerateSequence retorna um canal somente-leitura contendo 'n' inteiros.
func GenerateSequence(n int) <-chan int {
	// 1. Criação do canal interno
	out := make(chan int)

	// 2. Despacho assíncrono do produtor
	go func() {
		// 3. Garantia de fechamento quando o gerador terminar
		defer close(out)

		for i := 1; i <= n; i++ {
			out <- i // 4. Envio do dado gerado
		}
	}()

	// 5. Retorno imediato do canal
	return out
}

func ExecuteBasicGenerator() {
	// Consumo idiomático com 'range'
	// O loop suspende até haver dados, e encerra sozinho quando 'out' for fechado.
	for num := range GenerateSequence(3) {
		fmt.Printf("Recebido: %d\n", num)
	}
}
```

**Implementação Passo a Passo:**

- **`<-chan int`:** A assinatura da função força a segurança de design. O chamador recebe um canal de onde _só pode ler_. Ele não pode enviar dados acidentalmente para este canal, prevenindo _Deadlocks_ e corrupção de estado.
    
- **`defer close(out)`:** Regra de ouro da concorrência: "Aquele que cria o canal e envia os dados é quem deve fechá-lo". Se o produtor não fechar o canal, o `for range` do consumidor no escopo principal tentará ler um quarto elemento que nunca chegará, resultando no _Panic: all goroutines are asleep - deadlock!_.
    

#### Tópico B: Padrão Sênior (Sinais de Aborto e Prevenção de Leaks)

Go

```
package concurrency

import (
	"context"
	"fmt"
	"time"
)

// InfiniteDataStream gera métricas infinitamente até ser cancelado.
func InfiniteDataStream(ctx context.Context) <-chan string {
	out := make(chan string)

	go func() {
		defer close(out)
		counter := 1

		for {
			payload := fmt.Sprintf("Métrica_%d", counter)

			// Multiplexação idiomática: tenta enviar ou aborta se cancelado
			select {
			case out <- payload:
				// Sucesso ao enviar para o consumidor
				counter++
				time.Sleep(10 * time.Millisecond) // Simula carga
				
			case <-ctx.Done():
				// O consumidor cancelou o contexto. 
				// Abortamos imediatamente a Goroutine.
				fmt.Println("[Gerador] Sinal de interrupção recebido. Desligando produtor.")
				return 
			}
		}
	}()

	return out
}

func ExecuteRobustGenerator() {
	// Cria um contexto que envia o sinal de cancelamento (Done) após uso
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel() // Garante limpeza da árvore de contextos

	stream := InfiniteDataStream(ctx)

	// Consumidor decide que só precisa dos 3 primeiros itens de um stream infinito
	for i := 0; i < 3; i++ {
		data := <-stream
		fmt.Println("Processando:", data)
	}

	// O consumidor sai do loop prematuramente.
	// O 'defer cancel()' da Main será acionado na saída da função, 
	// sinalizando via 'ctx.Done()' para a Goroutine parar de gerar dados.
}
```

**Implementação Passo a Passo:**

- **O Contexto como Parâmetro Primário:** Em APIs robustas, qualquer função que envolva latência de rede, concorrência ou loops infinitos deve aceitar um `context.Context` como primeiro parâmetro.
    
- **O Padrão Select-Send (`select { case out <- payload: ... case <-ctx.Done(): }`):** O operador `select` do Go avalia múltiplos canais simultaneamente. O produtor tenta injetar o `payload` no canal `out`. Se o consumidor parou de ler (pois já pegou os 3 itens que queria), o canal `out` fica cheio (bloqueado). Sem o `select`, a Goroutine travava aqui. Com o `select`, se a Main der o `cancel()`, o canal `ctx.Done()` recebe um sinal em _broadcast_. A instrução `select` desperta a rota `case <-ctx.Done()`, executa o `return`, o `defer close(out)` roda, e a Goroutine é destruída na memória com segurança.