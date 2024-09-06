# Apresentando Galexie: Extraia e armazene dados Stellar com eficiência

***Tradução***: [Arthur Fialho](www.linkedin.com/in/arthurfialho) Embaixador da Blockchain Stellar no Brasil.

**Artigo traduzido do site oficial da Stellar, link abaixo**

*Data da postagem 29/08/2024*

<https://stellar.org/blog/developers/introducing-galexie-efficiently-extract-and-store-stellar-data>

*Bem-vindo à ultima parcela da nossa série sobre a **Composable Data Platform**, a plataforma de acesso a dados de próxima geração na Stellar. Hoje estamos animados em apresentar o primeiro componente, Galexie, um aplicativo leve que extrai dados da *ledger* na rede Stellar. Continue lendo para saber mais!*

Um dos principais desafios em trabalhar com dados da rede Stellar é o acoplamento estreito entre os processos de extração e transformação de dados. Os desenvolvedores têm ferramentas limitadas disponíveis para ler e salvar dados diretamente na rede Stellar, o que limita a flexibilidade no design de seus aplicativos. Para resolver isso, criamos o Galexie como um aplicativo extrator dedicado, desacoplando a ingestão de dados da transformação. Essa abordagem oferece maior versatilidade em comparação à hospedagem de um serviço monolítico, como Horizon ou Hubble.

A função primária do Galexie é exportar eficientemente todo o corpus da rede Stellar e armazená-lo em um ambiente baseado em nuvem [**data lake**](https://aws.amazon.com/what-is/data-lake/). Isso cria uma base de dados pré-processados que podem ser rapidamente acessados e transformados por vários aplicativos personalizados adaptados a necesssidades específicas.

Os dados exportados pelo Galexie dão suporte a ferramentas como Hubble e Horizon na construção de dados derivados. Diferentemente dos *ledgers* brutos escritos nos [**Arquivos de história**](https://github.com/stellar/stellar-core/blob/master/docs/history.md), que é uma visão não processada do histórico da rede, o Galexie exporta metadados completos de transações, oferecendo dados pré-computados prontos para uso.

Este design não só suporta ferramentas existentes, mas também permite um processamento de dados mais eficiente e abre novas possibilidades para análise de dados dentro do ecossistema Stellar. Nas seções a seguir, exploraremos os recursos, a arquitetura e as aplicações potenciais do Galexie.

## O que é o Galexie?

Galexie é um aplicativo simples e leve que agrupa dados da rede Stellar, os processa e os grava em um armazenamento de dados externos. Ele inclui os seguintes recursos:

- Extração de dados: extrai metadados de transações da rede Stellar.
- Compactação: comprime os metadados da transação para armazenamento otimizado.
- Opções de armazenamento: conecte-se a um armazenamento de dados de sua escolha, começando com o Google Cloud Storage (GCS).
- Modos de operação: carregue um intervalo fixo da *ledger* para o armazenamento em nuvem ou transmita continuamente novas *ledgers* para o armazenamento em nuvem à medida que são gravados na rede Stellar.

## Visão geral da arquitetura

O Galexie é construído em torno de alguns princípios-chave de design:

- Simplicidade: Faz apenas uma coisa, exportar dados da *ledger* de forma eficiente, e faz muito bem.
- Portabilidade: Escreva dados de forma plana, para que não haja necessidade de mecanismos complexos de indexação no sistema de armazenamento. Isso nos permite trocar facilmente uma opção de armazenamento de dados por outra.
- Descentralização: incentiva os construtores a possuir e gerenciar seus próprios dados.
- Extensibilidade: Facilita a adição de suporte para novos sistemas de armazenamento.
- Resiliência: Torne possível que operações interrompidas sejam retomadas de forma limpa após uma reinicialização. Um operador deve ser capaz de executar várias instâncias para redundância sem conflito.

Não-objetivo: Galexie não foi criado ou projetado para ingestão ao vivo ou interação com a rede Stellar. O núcleo cativo é muito mais adequado para ingestão ao vivo.

A seguir, um diagrama de arquitetura que descreve os vários componentes dentro do Galexie.

![imagem_galexie_diagrama](https://cdn.sanity.io/images/e2r40yh6/production-i18n/e3514e1d9e10aee8b2943cd5b0a7ecb1a1e326bf-1498x310.png?w=1440&auto=format)

## Formato e armazenamento de dados

Assim como o núcleo da Stellar, Galexie emite [**metadados de transação**](https://github.com/stellar/stellar-xdr/blob/72868e9ca22a57eec9a21610c9df67e07705f8fb/Stellar-ledger.x#L607-L613) no formato XDR padrão. Este formato é compatível com sistemas existentes, como Horizon e Hubble, o que permite que os desenvolvedores troquem sem esforço a fonte de dados XRD de seus backends de aplicativos.

Decidimos oferecer suporte ao armazenamento de várias *ledgers* em cada arquivo exportado. Isso reduz o número de arquivos gravados em um armazenamento de dados e torna significamente mais eficiente o download de grandes intervalos de dados. Descobrimos durante a experimentação que o download em massa de dados para o mesmo intervalo de *ledger* é duas vezes mais rápido quando os arquivos contêm um pacote de 64 *ledgers* em comparação com os arquivos que contêm um único *ledger*.

Todos os arquivos enviados são compactados usando [***Zstandard***](https://facebook.github.io/zstd/) como algoritmo de compressão padrão para economizar largura de banda de upload e download, bem como espaço de armazenamento.

## Estrutura de arquivo e diretório

Para permitir uma organização maior dos dados, adicionamos suporte para particionar os dados por meio de uma estrutura de diretório. O número de *ledgers* por arquivo e o número de arquivos por partição são ambos configuráveis.

Decidimos codificar informações adicionais nos nomes de arquivos e diretórios. Esses nomes agem como um token de classificação e eliminam a necessidade de indexação separada. O formato do nome de arquivo incluí dois componentes:

- Cada nome de diretório contém o primeiro e o último número do *ledger* contido na partição, os nomes de arquivo contêm os números iniciais e finais do *ledger* contidos no arquivo.
- Além disso, os nomes de diretórios são prefixados com um token de classificação, que é a representação hexadecimal de (uint_max - *ledger* inicial da partição). O nome do arquivo é prefixado com a representação hexadecimal de (uint_max - *ledger* inicial do arquivo). Esses tokens agem como identificadores de classificação.

Esse esquema de nomenclatura garante que os arquivos e diretórios sejam classificados em ordem decrescente, com os *ledgers* mais recentes aparecendo primeiro, otimizando a recuperação dos *ledgers* mais recentes.

Por exemplo, usando 64.000 arquivos por diretório e 64 *ledgers* por arquivo, o diretório e o arquivo contando os *ledgers* 64065 a 64128 seriam nomeados:

- Nome do diretório: FFFF05FE-64001-128000/
- Nome do arquivo: 0xFFFF05BE-64065-64128.xdr.gz

## Melhorando a eficiência com Galexie

A abordagem de armazenamento de dados do Galexie melhora significamente a eficiência operacional e a escalabilidade das ferramentas Stellar. Em vez de depender do *Captive core* para gerar metadados de transação, essas ferramentas agora podem acessar metadados de transação pré-computados do armazenamento de dados. Essa abordagem melhora a velocidade de recuperação de dados em uma ordem de magnitude!

Além disso, ao remover a dependência do *Captive Core*, que é um componente de computação pesada, o Galexie reduz os requisitos e custos de hardware.

Aqui estão dois exemplos que demonstram a eficácia do uso dessa abordagem de armazenamento de dados.

## Exemplos de caso de uso:

### Hubble (Stellar-ETL)

Antes do Galexie, o Hubble, a *Stellar Analytics Platform*, usava o *Captive Core* para ingerir dados da rede. Para processamento em lote, isso consumia muitos recursos porque a fase de recuperação do *Captive Core* demorava mais do que ler e processar esse lote de dados da rede. Ao alavancar um *data lake*, o Hubble é desacoplado do *Captive Core*, o que significa que ele pode acessar metadados de transações diretamente do GCS sem incorrer no alto custo de inicializaçãao do *Captive Core*. Isso reduz significamente o tempo de execução e a sobrecarga.

![imagem_galexie_diagrama_2](https://cdn.sanity.io/images/e2r40yh6/production-i18n/3d3b9a05c59650c564c4a80cccb6e1aa79f6ebfb-1600x877.png?w=1440&auto=format)

### Horizon

O Horizon, o servidor da API Stellar, tem a capacidade de realimentar dados históricos dentro de intervalos de *ledgers* especificados usando uma instância interna do *Captive Core*. Para gerenciar grandes intervalos de *ledgers* com eficiência, o Horizon pode executar vários processos de reingestão simultaneamente, cada um acessando seu próprio *Captive Core*. No entanto, essa abordagem é altamente intensiva em recursos. Desacoplar a reingestão do Horizon do *Captive Core* permite a reingestão direta de dados do *data lake*, aumentando a velocidade e a escalabilidade.

![imagem_galexie_diagrama_3](https://cdn.sanity.io/images/e2r40yh6/production-i18n/1d2bfc5bc8efa3d0216d46b1d07227a0dbf5968d-1440x858.png?w=1440&auto=format)

## Executando seu próprio Galexie

Executar sua própria instância Galexie permite que você crie um *data lake* personalizado com dados brutos extraídos da *ledger*. Essa abordagem oferece alguns benefícios:

- Flexibilidade: personalize o esquema e a estratégia de retenção dos seus dados.
- Acessibilidade de dados aprimorada: obtenha acesso direto aos dados, eliminando a dependência do Horizon ou do Stellar Core.
- Escalabilidade: armazene todo o histórico da rede Stellar de forma econômica, com a capacidade de preencher ou realimentar o histórico conforme necessário.

Para obter instruções detalhadas sobre como instalar e executar o serviço, consulte o [**Guia de administração**](https://github.com/stellar/go/blob/master/services/galexie/README.md)

### Desempenho

Criamos um *data lake* usando Galexie e registamos as seguintes observações:

- Executar uma única instância do Galexie é recomendado apenas para ingestão de dados de *ponto para frente*. O preenchimento de todo o histórico da rede levaria 150 dias executando uma única instância.
- É recomendável executar várias instâncias paralelas para preencher um *data lake*.
    - Por exemplo, executamos mais de 40 instâncias paralelas do Galexie (em motores de computação [**e2-standard2**](https://cloud.google.com/compute/docs/general-purpose-machines#e2_machine_types_table)) ao criar um *data lake* de dados do *pubnet* demorou menos de 5 dias com um custo computacional estimado de US$600 para preencher 10 anos de dados.
- O tamanho do *data lake* para todo o histórico Stellar *pubnet* é de ~ 3TB
- O custo operacional de executar o Galexie para exportar continuamente dados da rede é de ~US$160 por mês. Isso inclui US$60 por mês em computação e outros US$100 por mês em custos de armazenamento.

Observação: as estimativas de tempo relacionadas ao preenchimento de um *data lake* dependem de vários fatores, como o número de registros agrupados em um único arquivo, latência da rede e largura de banda.

## O que vem a seguir

### Desenvolvimento Futuro

Nosso objetivo na SDF é avançar verdadeiramente sobre aplicativos descentralizados e construir componentes de software menores e reutilizáveis. O Galexie se alinha com essa visão, desempenhando um papel importante na construção da *Composable Data Platform*. Nosso objetivo é expandir a funcionalidade do Galexie adicionando suporte para outros armazenamentos de objetos e integrando com novas ferramentas de transformação.

### Como você pode contribuir

Agradecemos as contribuições da comunidade para dar suporte a esses desenvolvimentos futuros. Seu envolvimento pode ajudar a estender os recursos do Galexie, adicionar suporte para soluções adicionais de armazenamento em nuvem e integrar novas ferramentas. Para mais detalhes sobre como se envolver, consulte nosso [**Guia do desenvolvedor**](https://github.com/stellar/go/blob/master/services/galexie/DEVELOPER_GUIDE.md)

### O que está por vir?

Em postagens de blog futuras, focaremos em casos de uso específicos, incluindo a refatoração de backends do Hubble e do Horizon para usar um *data lake* gerado por Galexie. Fique ligado enquanto nos aprofundamos nessas aplicações e seus *benchmarks*, e fornecemos insights sobre como o Galexie pode aprimorar o manuseio e o desempenho de dados.

### Feedback

Se você tiver dúvidas, sugestões ou feedback, junte-se a nós em [**Discord**](https://discord.gg/stellardev) nos canais [**#hubble**](https://discord.com/channels/897514728459468821/1214961876253933649) e [**#horizon**](https://discord.com/channels/897514728459468821/912466080960766012). Nossa equipe e membros da comunidade estão lá oara discutir e fornecer suporte.