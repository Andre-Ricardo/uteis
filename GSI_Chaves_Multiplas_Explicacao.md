# Entendendo o Suporte Nativo a Múltiplas Chaves no GSI do DynamoDB com C#

Com o anúncio recente da AWS sobre o **suporte a múltiplas chaves (Multi-key support) em Global Secondary Indexes (GSIs)**, a forma de modelar dados para consultas complexas no DynamoDB mudou radicalmente para melhor.

Anteriormente, desenvolvedores precisavam usar táticas de modelagem como concatenar múltiplos atributos em um só (ex: criar um atributo extra `MarcaModelo = "Ford#Mustang"`) para simular uma chave composta, o que adicionava complexidade e duplicação de dados.

Agora, o DynamoDB permite nativamente que você especifique **até 4 chaves de partição (Partition Keys)** e **até 4 chaves de classificação (Sort Keys)** em um único GSI.

## Como funciona o Novo Suporte a Multi-key

Este novo recurso simplifica muito a modelagem e o código da aplicação, pois:
1. **Tipos de Dados Nativos**: Cada atributo mantém seu tipo de dado original (String, Number, Boolean), não havendo mais necessidade de converter tudo para String para fins de concatenação (ideal para trabalhar com Datas/Anos numéricos ou booleanos).
2. **Eficiência e Limpeza**: Não é necessário criar campos artificiais para servir de índice no seu Item, deixando a estrutura limpa.

### Regras para Querying com Múltiplas Chaves:
Segundo a documentação, para fazer as consultas nestes índices, você deve seguir regras simples:
- **Partition Keys**: Você deve fornecer a condição de **igualdade (`=`)** para **todas** as chaves de partição configuradas no GSI.
- **Sort Keys**: São opcionais, e você também pode usar igualdade (`=`) nelas. 
- **Operadores de Range**: Operadores de intervalo (`<`, `>`, `<=`, `>=`, `BETWEEN`, `BEGINS_WITH`) só são permitidos no **último** atributo de Sort Key informado na sua consulta.
- A ordem importa! Você não pode pular atributos da Sort Key na sua consulta (deve ser em sequência da esquerda para a direita).

---

## Como isso funciona na prática no C#

### 1. A Configuração da Tabela e do Índice

Você agora configura o seu GSI `MarcaModelo-Ano-Index` diretamente na nuvem AWS contendo:
- **Partition Keys**: `Marca` (String) e `Modelo` (String)
- **Sort Key**: `Ano` (Number)

### 2. Inserção Limpa no C#

Ao inserir dados, você não precisa mais se preocupar em montar campos concatenados sintéticos. Você salva o item no seu formato mais natural:

```csharp
var carro = new Carro
{
    Id = Guid.NewGuid().ToString(),
    Marca = "Ford",   // <-- PK 1 do GSI
    Modelo = "Mustang", // <-- PK 2 do GSI
    Ano = 2021,       // <-- SK 1 do GSI
    Cor = "Preto"
};
```

### 3. A Query no C# 

Graças a este novo recurso, a implementação que está no seu código `DynamoDbCarRepository.cs` é **nativamente suportada**, válida e extremamente elegante. Nós apenas passamos as chaves múltiplas unindo-as com a palavra-chave `AND` diretamente na string do `KeyConditionExpression`:

```csharp
public async Task<PagedResult<Carro>> BuscarCarrosAsync(string marca, string modelo, int anoInicio, int anoFim)
{
    var request = new QueryRequest
    {
        TableName = "Carros",
        IndexName = "MarcaModelo-Ano-Index", 
        
        // Expressão com múltiplas chaves (Duas PKs e uma SK) agora é totalmente válida e suportada!
        KeyConditionExpression = "Marca = :v_marca AND Modelo = :v_modelo AND Ano BETWEEN :v_anoInicio AND :v_anoFim",
        
        ExpressionAttributeValues = new Dictionary<string, AttributeValue>
        {
            { ":v_marca", new AttributeValue { S = marca } },
            { ":v_modelo", new AttributeValue { S = modelo } },
            // O tipo numérico (N) é suportado nativamente nas chaves de range!
            { ":v_anoInicio", new AttributeValue { N = anoInicio.ToString() } },
            { ":v_anoFim", new AttributeValue { N = anoFim.ToString() } }
        }
    };

    var response = await _dynamoDbClient.QueryAsync(request);
    // ... mapeamento da resposta ...
}
```

### Por que esta Query é válida com o novo recurso?
- `Marca = :v_marca` -> Condição de igualdade na PK 1.
- `Modelo = :v_modelo` -> Condição de igualdade na PK 2.
- `Ano BETWEEN :v_anoInicio AND :v_anoFim` -> Condição de *range* (intervalo) permitida na última Sort Key fornecida.

Isso atende perfeitamente a todas as restrições impostas pela nova funcionalidade da AWS.

## Resumo dos Benefícios
- **Menor complexidade de código**: Não precisamos escrever lógica de string splitting ou interpolação na aplicação antes de inserir ou consultar dados.
- **Tipos preservados**: Mantivemos o campo "Ano" como numérico para consultas robustas de `BETWEEN`.
- **Evolução do Schema**: Permite que você crie novos índices com os atributos naturais conforme as necessidades de negócio mudem.

## Como funciona a Paginação com GSIs no C#

A paginação de dados em consultas (Queries) do DynamoDB é extremamente eficiente e funciona utilizando um sistema de cursores com duas propriedades principais: `Limit` e `ExclusiveStartKey`. O seu código em `DynamoDbCarRepository.cs` já implementa de forma exata esse conceito.

### 1. Definindo o limite de itens por página (`Limit`)

Na requisição, você define a quantidade máxima de itens que deseja receber em uma única chamada usando a propriedade `Limit`. O DynamoDB processará a consulta até atingir a quantidade estipulada ou o limite de 1MB de dados processados por requisição, o que ocorrer primeiro.

### 2. O Cursor de Retorno (`LastEvaluatedKey`)

Ao retornar a resposta, se o DynamoDB detectar que há mais itens correspondentes à sua consulta além dos retornados na página atual, ele preencherá a propriedade `LastEvaluatedKey` na resposta (`response.LastEvaluatedKey`).
- Se este dicionário for **nulo ou vazio**, significa que você chegou à última página e não há mais resultados disponíveis.
- Se estiver **preenchido**, ele contém a "chave" identificadora exata do último item lido. Você deve guardar esse valor (por exemplo, retornando-o na sua API para ser armazenado no frontend ou cache).

### 3. Buscando a Próxima Página (`ExclusiveStartKey`)

Para pedir a "próxima página" de dados, você deve enviar um novo `QueryRequest` idêntico ao primeiro, porém injetando o valor guardado do `LastEvaluatedKey` dentro da propriedade `ExclusiveStartKey`. Assim o DynamoDB sabe exatamente de onde ele deve retomar a leitura.

Como exemplificado no seu repositório:

```csharp
var request = new QueryRequest
{
    // ... definições de Tabela, IndexName e KeyConditionExpression ...
    
    // Limita o número de resultados retornados na página
    Limit = limit, 
    
    // Injeta o "cursor" de onde a busca deve iniciar (se houver vindo da página anterior)
    ExclusiveStartKey = exclusiveStartKey 
};

var response = await _dynamoDbClient.QueryAsync(request);

var pagedResult = new PagedResult<Carro>
{
    // Capturamos e expomos o cursor retornado para que o chamador saiba como pedir a próxima página
    LastEvaluatedKey = response.LastEvaluatedKey, 
    Items = new List<Carro>()
};
```

Com este modelo de *keyset pagination* integrado, a busca em um índice global (GSI) com múltiplas chaves mantém sua performance consistente (O(1) para achar a partição e leitura contínua dos dados), lendo estritamente os dados necessários e reduzindo de forma drástica os custos com o faturamento da AWS.

### 4. O que acontece se o Limite (Limit) mudar entre as páginas?

O DynamoDB não se importa se você alterar o limite (`Limit`) de uma página para a outra durante a navegação. O funcionamento da paginação é estritamente baseado no "Cursor" (a chave informada).

**Exemplo:**
- O cliente pede a **Página 1** com `Limit = 10`. O banco retorna 10 carros e entrega um cursor apontando para o último carro lido (carro de número 10).
- Em seguida, o usuário quer ver uma grade maior de resultados. Ele pede a **Página 2** passando o **mesmo cursor** recebido, mas com a propriedade `Limit = 50`.
- O DynamoDB respeitará o limite atual e, partindo exatamente de onde o cursor parou (carro 10), buscará até 50 novos registros. Se encontrar, ele entregará a resposta e um novo cursor apontando para o último carro lido desta nova página (ex: carro 60).

Portanto, **a chave (cursor) exigida não muda se o limite mudar**. Você sempre deve enviar no `ExclusiveStartKey` exatamente o dicionário que recebeu no `LastEvaluatedKey` da chamada imediatamente anterior.

### 5. Exemplo Prático: Como a Paginação ficaria na Controller (ASP.NET Core API)

Como a propriedade `LastEvaluatedKey` é um dicionário complexo que depende da estrutura do DynamoDB (`Dictionary<string, AttributeValue>`), trafegar esse objeto cru em parâmetros de URL de uma API REST (`[FromQuery]`) é impraticável.

A prática altamente recomendada no C# é serializar esse Dicionário em formato JSON e convertê-lo para Base64. Dessa forma, o cliente (Front-end ou App) recebe e manipula apenas uma string simples ("Token de Paginação").

Veja um exemplo de como implementar isso numa `Controller`:

```csharp
using System.Text.Json;
using System.Text;
using Microsoft.AspNetCore.Mvc;
using Amazon.DynamoDBv2.Model; // Necessário para acessar as classes do AWS SDK

[ApiController]
[Route("api/[controller]")]
public class CarrosController : ControllerBase
{
    private readonly DynamoDbCarRepository _repository;

    public CarrosController(DynamoDbCarRepository repository)
    {
        _repository = repository;
    }

    [HttpGet]
    public async Task<IActionResult> GetCarros(
        [FromQuery] string marca, 
        [FromQuery] string modelo, 
        [FromQuery] int anoInicio, 
        [FromQuery] int anoFim, 
        [FromQuery] int limit = 10, 
        [FromQuery] string paginationToken = null) // O frontend envia a chave como String
    {
        Dictionary<string, AttributeValue> startKey = null;

        // 1. Conversão String (Token Base64) -> Dicionário AWS DynamoDB
        if (!string.IsNullOrEmpty(paginationToken))
        {
            var jsonKey = Encoding.UTF8.GetString(Convert.FromBase64String(paginationToken));
            startKey = JsonSerializer.Deserialize<Dictionary<string, AttributeValue>>(jsonKey);
        }

        // 2. Chama o seu repositório perfeitamente limpo (O Limite e o Cursor são passados para a API AWS)
        var pagedResult = await _repository.BuscarCarrosAsync(marca, modelo, anoInicio, anoFim, limit, startKey);

        // 3. Conversão Dicionário AWS DynamoDB -> String (Token Base64)
        string nextToken = null;
        if (pagedResult.LastEvaluatedKey != null && pagedResult.LastEvaluatedKey.Count > 0)
        {
            var jsonNextKey = JsonSerializer.Serialize(pagedResult.LastEvaluatedKey);
            nextToken = Convert.ToBase64String(Encoding.UTF8.GetBytes(jsonNextKey));
        }

        // 4. Retorna os dados mapeados para o cliente (mantendo o backend desacoplado do DB)
        return Ok(new 
        {
            Items = pagedResult.Items,
            NextPaginationToken = nextToken // Se for null, o frontend saberá que as páginas acabaram
        });
    }
}
```

**Por que fazer isso na Controller?**
Com essa abordagem, a sua API fica abstraída. O Front-end não tem ideia de que você usa o DynamoDB nos bastidores, ele não lida com chaves `AttributeValue`. Ele apenas sabe que para pegar a próxima página ele deve enviar a requisição usando `?limit=50&paginationToken=O_TOKEN_RETORNADO_NO_BODY`.
