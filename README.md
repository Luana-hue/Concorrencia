# LABORATÓRIO DE INVESTIGAÇÃO: A CONDIÇÃO DE CORRIDA

## A BATALHA DO SPOOLER DE IMPRESSÃO

### Equipe 3 — Concorrência

---

## 1. DIAGNÓSTICO E CAUSA RAIZ

O problema identificado pelo nosso grupo é uma **condição de corrida** no sistema que organiza a fila de impressão, chamado **spooler**.

Em uma gráfica rápida, múltiplos terminais enviam arquivos pesados para uma impressora 3D industrial. Ocasionalmente, o equipamento imprime a primeira camada do projeto de um cliente e a segunda camada do projeto de outro, causando a fusão indesejada de peças.

Isso acontece porque vários processos acessam e alteram os mesmos dados sem um controle adequado de sincronização.

Nas páginas **82 e 83** de *Sistemas Operacionais Modernos*, Tanenbaum e Bos apresentam um exemplo de spool de impressão no qual dois processos podem escolher a mesma posição da fila. Os dois processos consultam o mesmo valor antes que ele seja atualizado e, como consequência, um processo pode sobrescrever a informação registrada pelo outro. Esse comportamento caracteriza uma **condição de corrida** (TANENBAUM; BOS, 2016, p. 82-83).

No caso analisado pelo grupo, ocorre uma situação semelhante. Os terminais de impressão acessam simultaneamente o diretório de *spooling*, que funciona como uma área compartilhada para armazenar temporariamente os arquivos que serão processados pela impressora.

O problema ocorre porque dois ou mais processos podem consultar e modificar simultaneamente os dados responsáveis pelo controle da fila. Dessa forma, um processo pode iniciar a gravação ou alteração de um arquivo enquanto outro processo também está realizando uma operação sobre o mesmo recurso.

O resultado é a mistura das informações dos projetos, fazendo com que diferentes camadas de arquivos G-code sejam associadas incorretamente durante a impressão 3D.

Portanto, a **causa raiz** do problema é a ausência de sincronização adequada no acesso ao diretório de impressão e às variáveis de controle compartilhadas.

---

## 2. FALHA DE ATOMICIDADE

Além da condição de corrida, o problema apresenta uma falha de **atomicidade**.

Uma operação atômica deve ser realizada como uma unidade indivisível, de forma que outro processo não consiga interferir no meio da execução da operação.

No caso do spooler, a operação de inserir um novo trabalho na fila deveria ocorrer de maneira indivisível. O processo deveria:

1. verificar a posição disponível;
2. registrar o arquivo de impressão;
3. atualizar o controle da fila;
4. finalizar a operação.

Outro processo não deveria conseguir interferir entre essas etapas.

Quando isso não acontece, dois processos podem observar o mesmo estado do sistema e tentar utilizar o mesmo recurso simultaneamente. Consequentemente, um arquivo pode substituir ou interferir nos dados do outro.

Esse comportamento explica a situação apresentada no estudo de caso, na qual camadas de projetos diferentes acabam sendo processadas como se pertencessem ao mesmo trabalho de impressão.

O conceito de operação atômica é importante para garantir que uma operação compartilhada seja concluída de maneira consistente, sem sofrer interferência de outro processo durante sua execução.

---

## 3. REGIÃO CRÍTICA

O trecho de código responsável pela consulta do slot disponível, gravação do arquivo e atualização dos ponteiros do diretório de *spooling* constitui uma **Região Crítica**.

De acordo com Tanenbaum e Bos, uma região crítica é a parte do programa em que ocorre acesso a recursos compartilhados e que, portanto, precisa ser protegida contra acessos simultâneos de processos concorrentes (TANENBAUM; BOS, 2016, p. 83).

No cenário analisado, a região crítica envolve principalmente as operações realizadas sobre:

- o diretório de *spooling*;
- a posição disponível na fila;
- o arquivo de impressão;
- os ponteiros ou variáveis de controle da fila;
- os dados utilizados para organizar os trabalhos enviados à impressora.

É justamente nesse trecho que ocorre a possibilidade de interferência entre processos.

Sem um mecanismo de proteção, dois terminais podem acessar a mesma posição da fila ao mesmo tempo, fazendo com que um trabalho seja sobrescrito ou combinado incorretamente com outro.

---

## 4. REQUISITOS PARA A SOLUÇÃO DA REGIÃO CRÍTICA

Para solucionar o problema, o mecanismo utilizado deve atender aos critérios fundamentais de exclusão mútua apresentados por Tanenbaum e Bos nas páginas **83 e 84**.

### 4.1 Exclusão Mútua

Apenas um processo pode estar dentro da região crítica por vez.

No spooler, isso significa que somente um terminal poderá realizar simultaneamente as operações de consulta, gravação e atualização do recurso compartilhado.

### 4.2 Independência de Hardware

A solução não deve depender da velocidade relativa dos processos nem da quantidade de CPUs existentes no sistema.

O mecanismo de sincronização deve funcionar corretamente independentemente da configuração do hardware.

### 4.3 Ausência de Bloqueio Indevido

Um processo que esteja executando fora de sua região crítica não deve impedir que outro processo consiga entrar nela.

Isso evita que um processo sem necessidade de acessar o spooler impeça os demais de utilizarem o recurso.

### 4.4 Ausência de Starvation

Nenhum processo deve esperar indefinidamente para acessar a região crítica.

Todos os processos que necessitam utilizar o spooler devem ter oportunidade de obter acesso ao recurso em algum momento.

---

## 5. SOLUÇÃO PROPOSTA: MECANISMO MUTEX

Para solucionar a falha de concorrência, propõe-se a utilização de um **Mutex** (*Mutual Exclusion*).

O Mutex funciona como um mecanismo de bloqueio que garante que apenas um processo por vez possa acessar a região crítica.

No sistema de impressão, o Mutex deve proteger o trecho responsável pela manipulação do diretório de *spooling*.

O funcionamento pode ser representado da seguinte maneira:

```text
Processo solicita acesso ao spooler
            ↓
       Mutex disponível?
          ↙       ↘
       SIM         NÃO
        ↓           ↓
 Adquire Mutex    Aguarda
        ↓
 Entra na Região Crítica
        ↓
Consulta a fila
        ↓
Grava o arquivo
        ↓
Atualiza os dados
        ↓
Sai da Região Crítica
        ↓
Libera o Mutex
        ↓
Próximo processo pode acessar
```

Dessa forma, quando um terminal estiver modificando o diretório de impressão, os demais processos deverão aguardar a liberação do Mutex.

Depois que o primeiro processo terminar todas as operações necessárias, o recurso será liberado e poderá ser utilizado pelo próximo processo.

Essa abordagem impede que dois processos modifiquem simultaneamente os mesmos dados e evita que os arquivos G-code sejam misturados.

---

## 6. FUNCIONAMENTO DA SOLUÇÃO NO CASO DA IMPRESSORA 3D

Com a implementação do Mutex, o processamento dos arquivos ocorre de maneira controlada.

Quando o **Terminal A** solicita acesso ao diretório de impressão, ele adquire o Mutex e entra na região crítica.

Enquanto o Terminal A estiver:

- verificando a posição disponível;
- gravando o arquivo;
- atualizando o controle da fila;

o **Terminal B** não poderá modificar o mesmo recurso.

O Terminal B ficará aguardando até que o Terminal A termine sua operação e libere o Mutex.

Após a liberação, o Terminal B poderá entrar na região crítica e realizar sua própria operação.

Assim, os arquivos são registrados separadamente e a ordem de processamento da fila permanece consistente.

O resultado esperado é que cada projeto seja processado de forma independente, eliminando a possibilidade de uma camada de um projeto ser combinada com a camada de outro.

---

## 7. PESQUISA MULTIMÍDIA E FUNDAMENTAÇÃO

Para complementar o estudo de caso **“A Batalha do Spooler de Impressão”**, foram utilizados materiais em três formatos: **texto, vídeo e áudio**.

A pesquisa teve como foco os conceitos de:

- condição de corrida;
- região crítica;
- exclusão mútua;
- concorrência;
- sincronização entre processos;
- atomicidade.

### 7.1 Fonte Textual

Como principal fonte teórica foi utilizado o livro *Sistemas Operacionais Modernos*, de Andrew S. Tanenbaum e Herbert Bos.

Nas páginas **82 e 83**, os autores apresentam um exemplo relacionado ao *spooling* de impressão em que dois processos tentam utilizar o mesmo diretório compartilhado. Os dois processos podem identificar a mesma posição livre e, dependendo da ordem de execução, um deles pode sobrescrever a informação registrada pelo outro.

Esse exemplo demonstra diretamente uma **condição de corrida** e possui relação com o problema apresentado no estudo de caso da gráfica (TANENBAUM; BOS, 2016, p. 82-83).

Na página **83**, o livro também apresenta o conceito de **região crítica** e explica a necessidade de impedir que dois processos acessem e alterem simultaneamente um mesmo recurso compartilhado. A **exclusão mútua** é apresentada como mecanismo para controlar esse acesso, permitindo que apenas um processo utilize a região crítica por vez (TANENBAUM; BOS, 2016, p. 83).

Na página **84**, são apresentados critérios e mecanismos relacionados à exclusão mútua, contribuindo para a fundamentação da solução proposta para o problema do spooler (TANENBAUM; BOS, 2016, p. 84).

A página **89** também é utilizada como fundamentação para o conceito de **atomicidade**, necessário para compreender por que as operações de atualização do spooler devem ocorrer sem interferência de outros processos durante sua execução (TANENBAUM; BOS, 2016, p. 89).

---

### 7.2 Fonte em Vídeo

Como material em vídeo, foi utilizada uma aula da disciplina de **Sistemas Operacionais da Escola Politécnica da Universidade de São Paulo (USP)**, apresentada pelo professor **João José Neto**.

A aula aborda temas relacionados à administração de processos, comunicação e sincronização de processos concorrentes.

O conteúdo complementa a fundamentação teórica do trabalho ao apresentar conceitos relacionados à execução de processos e à necessidade de mecanismos de sincronização quando diferentes processos trabalham de forma concorrente.

---

### 7.3 Fonte em Áudio

Como material em áudio, foi utilizado o episódio **“#167 Threads, Paralelismo e SO na Prática para Devs”**, do podcast **Café Debug**.

O episódio aborda conceitos relacionados a *threads*, concorrência, paralelismo e troca de contexto.

Esse conteúdo contribui para compreender por que execuções concorrentes precisam de mecanismos de controle quando diferentes processos ou *threads* compartilham recursos.

---

## 8. INSTRUMENTO VISUAL DE SÍNTESE

O grupo desenvolveu um **diagrama de síntese** utilizando o draw.io para representar visualmente o problema e a solução proposta.

O diagrama apresenta o fluxo desde os terminais de impressão até o diretório do spooler, destacando:

- Terminal A;
- Terminal B;
- Diretório do spooler;
- Condição de corrida;
- Região crítica;
- Exclusão mútua;
- Resultado correto.

A representação demonstra que os dois terminais acessam o mesmo diretório e que, sem sincronização, ocorre uma condição de corrida.

A aplicação da exclusão mútua garante que somente um processo por vez tenha acesso à região crítica, permitindo que os arquivos sejam processados separadamente.

### Diagrama

**Instrumento visual produzido no draw.io:**

<img width="277" height="523" alt="image" src="https://github.com/user-attachments/assets/d043de68-d8e2-4bcf-862a-0d5e6253fdac" />


---

## 9. CONCLUSÃO

A análise do caso **“A Batalha do Spooler de Impressão”** permitiu identificar uma falha de concorrência causada pelo acesso simultâneo e não sincronizado ao diretório de *spooling*.

A causa principal do problema é uma **condição de corrida**, na qual dois ou mais processos podem acessar e modificar os mesmos dados compartilhados simultaneamente.

A situação também envolve uma falha de **atomicidade**, pois as operações de inserção e atualização dos trabalhos de impressão podem ser interrompidas ou sofrer interferência de outro processo antes de serem concluídas.

O trecho responsável pelo acesso e alteração do diretório de impressão caracteriza uma **região crítica** e, por isso, precisa ser protegido por um mecanismo de exclusão mútua.

A solução proposta é a utilização de um **Mutex**, garantindo que apenas um processo por vez tenha acesso à região crítica. Dessa maneira, cada arquivo é registrado e processado separadamente, evitando a mistura das camadas dos projetos.

Portanto, a aplicação da exclusão mútua permite recuperar a consistência do sistema de impressão e solucionar a vulnerabilidade de concorrência apresentada no estudo de caso.

---

# REFERÊNCIAS

TANENBAUM, Andrew S.; BOS, Herbert. **Sistemas operacionais modernos**. 4. ed. São Paulo: Pearson Education do Brasil, 2016.

NETO, João José. **PCS 3446 – Sistemas Operacionais – aula 09**. São Paulo: Escola Politécnica da Universidade de São Paulo, [s. d.]. Vídeo. Disponível em: https://eaulas.usp.br/portal/home/video.action?idItem=18843. Acesso em: 25 out. 2026.

CAFÉ DEBUG. **#167 Threads, Paralelismo e SO na Prática para Devs**. 14 jul. 2025. Podcast, 1 h 08 min. Disponível em: https://podcasts.apple.com/br/podcast/167-threads-paralelismo-e-so-na-pr%C3%A1tica-para-devs/id1367730836?i=1000717118967. Acesso em: 25 out. 2026.
