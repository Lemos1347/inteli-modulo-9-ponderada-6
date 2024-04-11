# Inteli Módulo 9 - ponderada 6

## Atividade

Complete o [desafio do arquivo de 1 bilhão de linhas](https://1brc.dev/). Só isso. Sem restrição de performance, só complete o desafio.

Pode usar Python com numpy? Pode.

Pode transformar o arquivo em Parquet? Pode.

Pode usar Go/Rust/OCAML/Zig? Pode.

Pode usar uma solução publica para o desafio? Desde que consiga explicar, pode.

> [!IMPORTANT]
> Atenção! O código dessa solução é de autoria de [candrewlee14](https://github.com/candrewlee14/1brc-zig)

## Solução

Dentro da pasta [src](./src/) podem ser encontrados os seguintes arquivos:

- [create-sample.c](./src/create-sample.c) - Código para criar um arquivo de 1 bilhão de linhas

- [main.zig](./src/main.zig) - Código para resolver o desafio

> [!IMPORTANT]
> Para compilar o código em Zig, é necessário ter o compilador instalado. Para instalar o compilador, siga as instruções em [ziglang.org](https://ziglang.org/). E não precisa se preocupar com o código em C que o Zig também o compilará.

Para compilar o código em Zig, execute o seguinte comando na root desse projeto:

```bash
  zig build -Doptimize=ReleaseFast
```

Logo em seguida, você verá as pastas: `zig-cache/` e `zig-out/`. Os arquivos executáveis estarão dentro da pasta `zig-out/`.

Primeiro é necessário criar o arquivo de 1 bilhão de linhas. Para isso, execute o seguinte comando:

```bash
  ./zig-out/bin/run-create-sample 1000000000
```

E agora é só executar o código principal (_obs: é bom rodar com o time para sabermos quanto tempo levou a solução_):

```bash
  time ./zig-out/bin/1brc-zig measurements.txt
```

### Explicação da solução

O programa Zig fornecido é projetado para processar eficientemente um arquivo de texto grande que contém medições de temperatura de várias estações meteorológicas. O arquivo tem 1 bilhão de linhas, e o objetivo é calcular a temperatura mínima, média e máxima para cada estação. A solução utiliza uma abordagem multithreaded para lidar com o grande volume de dados de maneira eficiente.

#### Estruturas de Dados Principais

- **Stat**: Esta estrutura armazena estatísticas de temperatura (mínima, máxima, soma e contagem) para cada estação. Ela possui métodos para adicionar um item (`addItem`) e para fundir estatísticas de dois objetos `Stat` (`mergeIn`).

- **WorkerCtx**: Representa o contexto de um trabalhador (thread). Contém um mapa (`std.StringHashMap`) para armazenar as estatísticas (`Stat`) para cada estação e uma lista (`std.ArrayList`) para manter as estações únicas processadas por essa thread.

#### Processamento de Linhas

- **Divisão do Arquivo em Chunks**: O arquivo é mapeado na memória e dividido em pedaços (chunks) que são processados em paralelo. Cada thread processa um chunk, reduzindo o overhead de leitura do arquivo e permitindo que a computação seja distribuída entre diferentes CPUs.

1. Tamanho do Arquivo: O programa primeiro determina o tamanho total do arquivo em bytes, não em número de linhas.
2. Número de Threads: O programa determina o número de threads a ser usado como o número de CPUs disponíveis menos um. Essa escolha é feita para deixar um núcleo livre e evitar a sobrecarga total do sistema, especialmente em sistemas com um único núcleo. No entanto, isso é ajustável conforme a arquitetura e as necessidades específicas do sistema.
3. Divisão do Arquivo em Chunks: O arquivo é dividido em pedaços com base no número de bytes. Por exemplo, se o arquivo tem 10 GB (aproximadamente 10 bilhões de bytes) e há 7 threads disponíveis, cada chunk terá cerca de 1,43 GB (10 bilhões de bytes divididos por 7). Essa divisão não é feita por linhas, mas por bytes. O programa ajusta as fronteiras de cada chunk para garantir que terminem em uma quebra de linha, evitando assim que uma linha seja processada por duas threads diferentes.
   - _MMAP_: É utilizado a função mmap para mapear um arquivo em uma região da memória. Essa atividade não carrega o arquivo inteiro diretamente para a RAM no sentido tradicional. Em vez disso, ele cria uma espécie de "janela" na memória que reflete o conteúdo do arquivo. Isso é feito no nível do sistema operacional, que gerencia a relação entre a região de memória mapeada e o conteúdo do arquivo no disco.
   - _Vantagens MMAP:_ Quando é mapeado um arquivo em memória, o sistema operacional não carrega imediatamente todo o conteúdo do arquivo para a RAM. O carregamento das páginas de memória ocorre sob demanda, ou seja, somente as partes do arquivo acessadas são carregadas na RAM. Esse processo é conhecido como "page faulting". O sistema operacional cuida de carregar as partes (páginas) do arquivo que são necessárias para a RAM quando elas são acessadas e pode descarregá-las quando não são mais necessárias ou quando a memória é necessária para outros propósitos. Esse gerenciamento é transparente para o programa.
   - _Quantidade de jobs:_ O loop `for (0..job_count) |i| {...}` itera com base na quantidade de tarefas (job_count) que foi definida anteriormente (`CPU count`).
   - _Cálculo do Início da Pesquisa para Cada Chunk_: `const search_start = mapped_mem.len / job_count * (i + 1);` calcula onde a busca deve começar para delimitar o fim do chunk atual. Ele divide o comprimento total do mapeamento (`mapped_mem.len`) pelo número total de jobs (`job_count`) e multiplica pelo índice do job atual incrementado (`(i + 1)`), determinando assim a posição aproximada do fim do chunk.
   - _Encontrando o Final Real do Chunk:_ `const chunk_end = std.mem.indexOfScalarPos(u8, mapped_mem, search_start, '\n') orelse mapped_mem.len;` localiza a posição do próximo caractere de nova linha ('\n') após search_start. Isso é feito para garantir que o chunk termine em uma quebra de linha completa, evitando dividir uma linha de dados pela metade entre dois chunks. Se não encontrar uma nova linha, o fim do chunk é definido como o fim do arquivo (`mapped_mem.len`).
   - _Definindo o Chunk:_ `const chunk: []const u8 = mapped_mem[chunk_start..chunk_end];` cria uma slice (chunk) do mapeamento de memória que começa em chunk_start (início do chunk atual) e vai até chunk_end (fim do chunk), delimitando assim o pedaço do arquivo que a thread atual irá processar.
   - _Atualizando chunk_start para a Próxima Iteração:_ `chunk_start = chunk_end + 1;` prepara chunk_start para ser o início do próximo chunk na próxima iteração, movendo-o para a posição logo após o final do chunk atual.
   - _Iniciando e Distribuindo o Trabalho:_ `wg.start();` sinaliza o início de um novo trabalho. `try tp.spawn(threadRun, .{ chunk, i, &main_ctx, &main_mutex, &wg });` cria uma nova thread (ou utiliza uma do pool) para executar a função `threadRun`, passando o chunk e outras informações relevantes como argumentos. Essa função é responsável por processar o chunk designado.
   - _Condição de Parada:_ `if (chunk_start >= mapped_mem.len) break;` encerra o loop se o início do próximo chunk ultrapassar o tamanho do arquivo, garantindo que o programa não tente processar além do final do arquivo mapeado.
   - _obs.: \_O loop for em si não sobrecarrega a RAM porque não está duplicando o conteúdo do arquivo em memória; ele está apenas criando "vistas" (slices)_ _diferentes do mesmo mapeamento em_ _memória_ (`mapped_mem`)_. Essas vistas são passadas_ _para as várias threads para processamento_
4. Processamento Paralelo: Cada thread processa seu respectivo chunk, identificando linhas completas, extraindo dados e calculando as estatísticas necessárias. A busca por quebras de linha ('\n') ajuda a definir onde cada chunk começa e termina, garantindo que as linhas não sejam divididas entre diferentes threads.
   - Há uma função para delimitar o que cada thread vai fazer: `threadRun`. Nela há um loop `while (pos < chunk.len) {...}`, o qual percorre o chunk de bytes atribuído a essa thread, processando cada linha (registro) encontrada. `std.mem.indexOfScalarPos(u8, chunk, pos, ';')` encontra a posição do delimitador (;), que separa o nome da cidade da medição de temperatura na linha. `const city = chunk[pos..new_pos];` extrai o nome da cidade da linha atual. `const num = parseSimpleFloat(chunk, &pos);` chama parseSimpleFloat (uma função personalizada para realizar o parse de floats) para extrair e converter o valor da temperatura da linha, atualizando a posição (pos) para o início da próxima linha após o processamento.
   - A função personalizada de parseSimpleFloat percorre até 6 caracteres a partir da posição atual (pos) no chunk: `for (0..6) |i| {...}.` Esse limite assume que o número inclui um sinal opcional, até quatro dígitos e um ponto decimal, ou alguma combinação similar que não exceda seis caracteres. Dentro do loop, cada caractere é avaliado: 1. Se um '-' é encontrado, is_neg é definido como true, indicando um número negativo. 2. Se um dígito ('0'...'9') é encontrado, o dígito é convertido para um inteiro e acumulado em inum. A conversão é feita subtraindo '0' do caractere (aproveitando a tabela ASCII), e inum é atualizado multiplicando o valor atual por 10 antes de adicionar o novo dígito, deslocando assim o valor corretamente para a esquerda. 3. Se uma quebra de linha ('\n') é encontrada, isso indica o fim do número, e pos é atualizado para apontar para o próximo caractere após a quebra de linha, finalizando o loop. Após a saída do loop, inum é ajustado para refletir o sinal: `inum *= if (is_neg) -1 else 1;`. Se is_neg for verdadeiro, inum é multiplicado por -1. O valor inteiro inum é então convertido para um tipo de ponto flutuante (f32): `const num: f32 = @as(f32, @floatFromInt(inum)) / 10;`. A divisão por 10 é usada para posicionar o ponto decimal corretamente, assumindo que o número tinha uma casa decimal (mas não foi explicitamente separado por um ponto no formato de texto).
   - A atualização de estatísticas locais é atualizada. A estrutura de dados ctx.map mapeia cada cidade a suas estatísticas. Quando uma nova medição é encontrada, ela é adicionada ao registro da cidade correspondente usando addItem ou inicializa um novo registro com Stat{ .min = num, .max = num, .sum = num, .count = 1 }.
   - Após processar todo o seu chunk, a thread itera sobre as cidades em seu contexto local e mescla esses dados no contexto global (main_ctx), protegido por um mutex (main_mutex) para evitar condições de corrida. Se a cidade já existe no contexto global, suas estatísticas são mescladas usando mergeIn. Caso contrário, a cidade e suas estatísticas são adicionadas ao contexto global.
5. Fusão de Estatísticas: Após o processamento de todos os chunks, as estatísticas locais de cada thread são mescladas no contexto global. Isso é feito para garantir que todas as medições de temperatura para uma determinada cidade sejam agregadas corretamente, independentemente de qual thread as processou. O mutex é usado para garantir que a mesclagem seja feita de forma segura, evitando condições de corrida.
   - `std.mem.sortUnstable([]const u8, main_ctx.countries.items, {}, strLessThan);` é chamado para ordenar os nomes das estações meteorológicas (ou países, conforme o contexto dos dados) alfabeticamente. A função sortUnstable é utilizada aqui, ela é eficiente mas não garante estabilidade (isto é, elementos iguais podem ter sua ordem inicial alterada). A comparação é feita pela função strLessThan, que compara duas strings e determina a ordem entre elas.
