# Performance Report — Portal Engaja Brasil Conselheiros

> Medição realizada em: 2026-04-20  
> Arquivos analisados: `index.php` e `conselheiros_home.php`

---

## 1. Timeline Completa

| Etapa | Tempo acumulado | Tempo gasto na etapa |
|---|---|---|
| antes de inicial.php | 0.000 s | — |
| depois de inicial.php | 0.001 s | **0.001 s** |
| depois de funcoes.inc.php | 0.001 s | **0.000 s** |
| depois de PDO connect | 0.005 s | **0.004 s** |
| antes de topo.php | 0.007 s | 0.002 s |
| **depois de topo.php** | **4.765 s** | **🔴 4.758 s** |
| depois de PDO connect (conselheiros) | 4.769 s | 0.004 s |
| depois SQL cad_invest | 4.770 s | **0.001 s** |
| **depois API Contatos** | **6.426 s** | **🟠 1.656 s** |
| **depois API SalesOrder** | **9.281 s** | **🔴 2.855 s** |
| API Accounts (em andamento...) | 9.281 s+ | 🔴 em medição |

---

## 2. Distribuição do Tempo

```
topo.php (menu)     ████████████████████████████████████  4.758 s  (51%)
API SalesOrder      ██████████████████████               2.855 s  (31%)
API Contatos        ████████████                         1.656 s  (18%)
API Accounts        (sem valor final registrado)
Todo o resto        ░                                    0.012 s  (<1%)
```

---

## 3. Problemas Identificados

---

### 🔴 CRÍTICO — `topo.php` consome 4.758 s

**Arquivo:** `index.php` → `include("topo.php")`

O topo carrega um dos três menus dependendo da sessão:
- `menu_sys_invest.php` — para investidores logados
- `menu_sys_invest2.php` — comentado
- `menu_sys.php` — para não logados

O menu menu_sys_invest que é utilizado nos testes, faz chamadas para:

- 'mode' => 'Accounts','action'=>'resgatar'
- 'mode' => 'SalesOrder','action'=>'listar_registro'

```php
// No login — salvar na sessão
$_SESSION['nome_usuario'] = $obj[0]['nome'];
$_SESSION['plano_usuario'] = $obj[0]['id_plano'];

// No topo.php — usar da sessão, sem chamar API
echo $_SESSION['nome_usuario'];
```

---

### 🔴 CRÍTICO — API SalesOrder sequencial consome 2.855 s

**Arquivo:** `conselheiros_home.php`

```php
// executa sozinha, bloqueia até terminar
$result2c = curl_exec($ch2c); // SalesOrder — 2.855 s
```

**Solução recomendada:** Paralelizar as 3 chamadas com `curl_multi` (ver seção 5).

---

### 🟠 ALTO — API Contatos consome 1.656 s

**Arquivo:** `conselheiros_home.php`

```php
$result2c = curl_exec($ch2c); // Contatos — 1.656 s
```

Mesma causa: chamada sequencial e bloqueante. Resolvida junto com SalesOrder via `curl_multi`.

---

### 🟠 ALTO — API Accounts (sem tempo final registrado)

**Arquivo:** `conselheiros_home.php`

O log foi cortado antes do resultado, mas com base nas medições anteriores essa chamada custa entre **1.5 s e 2.7 s**.

---

### 🟡 MÉDIO — Três chamadas ao CRM sequenciais

**Arquivo:** `conselheiros_home.php`

As três APIs são independentes entre si — nenhuma depende do resultado da outra para ser disparada — mas estão sendo executadas uma após a outra:

```
API Contatos   →→→ espera →→→ API SalesOrder   →→→ espera →→→ API Accounts
   1.6 s                          2.8 s                          ~2.0 s
```

**Tempo atual (soma):** ~6.5 s  
**Tempo possível (paralelo):** ~2.8 s (tempo da mais lenta)  
**Economia estimada:** ~3.7 s

---

### 🟡 MÉDIO — SQL `cad_invest` redundante por página

**Arquivo:** `conselheiros_home.php`

```php
$f_rs = $con->prepare("select accountid from cad_invest where contactid = :pVariavel");
```

Essa query busca o `accountid` do conselheiro a cada carregamento. O resultado não muda entre páginas e poderia ser salvo na `$_SESSION` no login, eliminando a query.

```php
// Executar apenas uma vez, no login:
$_SESSION['id_cont_logado'] = $f_row->accountid;

// No conselheiros_home.php — apenas ler da sessão:
// (a query já foi removida no arquivo atual, mas o session
//  precisa ser populado em outro ponto)
```

---

### 🟢 BAIXO — `$_SESSION['tken']` gerado em toda requisição

**Arquivo:** `index.php`

```php
$_SESSION['tken'] = generateRandomString(24);
```

Isso recria o token a cada page load, o que pode causar problemas de validade em uploads simultâneos. Deveria ser gerado apenas quando o módulo `cad_comp_ong` for carregado (já existe essa checagem mais abaixo, mas a geração no topo é desnecessária).

---

### 🟢 BAIXO — Bloco `else` de `$homenova` nunca executa

**Arquivo:** `conselheiros_home.php`

```php
$homenova = 1; // hardcoded

if($homenova){ ... }
else{
    // ← este bloco NUNCA executa
    // contém 2 chamadas ONGs/resgatar e 1 Projetos/resgatar
    // código morto que pode ser removido
}
```

Não causa lentidão (não executa), mas aumenta o tamanho e a complexidade do arquivo desnecessariamente.

---

## 4. Resumo de Impacto

| Problema | Arquivo | Tempo desperdiçado | Prioridade |
|---|---|---|---|
| topo.php com chamada ao CRM | index.php | ~4.758 s | 🔴 Crítico |
| APIs sequenciais ao CRM | conselheiros_home.php | ~3.7 s ganho c/ paralelo | 🔴 Crítico |
| Dados do usuário não cacheados na sessão | index.php + topo.php | ~4.758 s (junto ao topo) | 🟠 Alto |
| SQL cad_invest por página | conselheiros_home.php | ~0.001 s | 🟡 Médio |
| Token gerado no topo sempre | index.php | desprezível | 🟢 Baixo |
| Bloco else código morto | conselheiros_home.php | 0 s (não executa) | 🟢 Baixo |

---



















# Melhorias de Desempenho — Página Home OSC (Engaja Brasil)


## 📊 Tabela de Tempos (Hard Refresh — sem cache)

| Cenário                                              | Tempo (s) |
|-----------------------------------------------------|----------|
| Com links estáticos                                 | 13       |
| Sem links estáticos                                 | 10       |
| Sem API (com links estáticos)                       | 06       |
| Sem API (sem links estáticos)                       | 05       |
| Com melhorias citadas no documento sem cache        | 07       |
| Com melhorias citadas no documento com cache        | 04       |

---

## 🧠 Leitura Rápida

- 🔗 Links estáticos adicionam ~3s
- 🌐 Chamadas de API adicionam ~4–7s
- ⚠️ Principal gargalo: dependência de API externa

---

## ⚠️ Diagnóstico Geral

- ❌ Sem cache
- ❌ Sem paralelismo
- ❌ Sem tratamento de falhas

➡️ Resultado:
- Alta latência
- Risco de travamento total se a API estiver lenta ou indisponível
- 
---

## Diagnóstico Geral

Sem cache, sem paralelismo e sem tratamento de falhas. Isso resulta em latência elevada e risco de travamento total da página caso o servidor externo esteja lento.

---

## 🟠 Alto — Queries repetidas no banco de dados

### Problema

Há múltiplas chamadas à função `RetornaDescNovo()` dentro de uma única condição, todas consultando a tabela `usuarios` com o mesmo critério (`id = $_SESSION['id_logado']`).

Isso gera diversas queries redundantes ao banco de dados, impactando negativamente o desempenho e aumentando o tempo de carregamento da aplicação.

```php
if (
    RetornaDescNovo('usuarios', 'nome', 'id', $_SESSION['id_logado'], $con) &&
    RetornaDescNovo('usuarios', 'fone1', 'id', $_SESSION['id_logado'], $con) &&
    RetornaDescNovo('usuarios', 'email', 'id', $_SESSION['id_logado'], $con) &&
    RetornaDescNovo('usuarios', 'cargo', 'id', $_SESSION['id_logado'], $con) &&
    RetornaDescNovo('usuarios', 'sobrenome', 'id', $_SESSION['id_logado'], $con)
)
```

### Impacto

* Múltiplas consultas desnecessárias ao banco
* Aumento no tempo de resposta
* Maior carga no servidor de banco de dados

### Sugestão de melhoria

Realizar apenas uma query para obter todos os dados necessários do usuário e armazená-los em variáveis ou estrutura (array/objeto), evitando chamadas repetidas:

```php
$usuario = BuscarUsuarioPorId($_SESSION['id_logado'], $con);

if (
    $usuario['nome'] &&
    $usuario['fone1'] &&
    $usuario['email'] &&
    $usuario['cargo'] &&
    $usuario['sobrenome']
)
```

---
## 🟠 Alto — Ausência de Cache

### Problema
Dados da ONG como `area_atuacao`, `porte_ong`, `status_ong` mudam raramente, mas são buscados na API a **cada acesso** à página.

### Solução com APCu (memória do servidor)

```php
$cache_key = 'ong_dados_' . $_SESSION['id_crm_logado'];
$cached = apcu_fetch($cache_key, $success);

if (!$success) {
    $result = fazChamadaCRM($data_ong);
    apcu_store($cache_key, $result, 300); // cache por 5 minutos
    $cached = $result;
}

$objc = json_decode($cached, true);
```

### Solução alternativa com arquivo (se APCu não estiver disponível)

```php
$cache_file = sys_get_temp_dir() . '/ong_' . $_SESSION['id_crm_logado'] . '.cache';

if (file_exists($cache_file) && (time() - filemtime($cache_file) < 300)) {
    $cached = file_get_contents($cache_file);
} else {
    $cached = fazChamadaCRM($data_ong);
    file_put_contents($cache_file, $cached);
}
```

**Impacto estimado:** elimina praticamente toda a latência da API em visitas repetidas.

---


## 🟠 Alto — Sem Timeout nas Chamadas cURL

### Problema
Se o servidor do CRM estiver lento ou indisponível, a página fica **travada indefinidamente** até o PHP atingir o timeout padrão (que pode ser de 30s ou mais).

### Solução

```php
curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, 3);  // máximo 3s para conectar
curl_setopt($ch, CURLOPT_TIMEOUT, 8);          // máximo 8s para resposta completa
```

Também é recomendado verificar erros após a execução:

```php
$result = curl_exec($ch);

if (curl_errno($ch)) {
    // log do erro, exibir mensagem amigável ao usuário
    error_log('cURL erro: ' . curl_error($ch));
    $result = null;
}
```

**Impacto estimado:** evita travamentos e melhora a experiência em cenários de instabilidade.

---

## 🟡 Médio — Chamadas cURL Sequenciais

### Problema
A chamada de **Projetos** só começa depois que a chamada de **ONGs** termina completamente, somando os tempos de forma linear.

```
[Chamada ONGs: ~300ms] → [Chamada Projetos: ~300ms] = ~600ms total
```

### Solução com `curl_multi`

```php
$mh = curl_multi_init();

$ch_ong = montaCurl($data_ong);
$ch_proj = montaCurl($data_projetos);

curl_multi_add_handle($mh, $ch_ong);
curl_multi_add_handle($mh, $ch_proj);

// Executa ambas em paralelo
do {
    $status = curl_multi_exec($mh, $running);
    curl_multi_select($mh);
} while ($running > 0);

$result_ong  = curl_multi_getcontent($ch_ong);
$result_proj = curl_multi_getcontent($ch_proj);

curl_multi_remove_handle($mh, $ch_ong);
curl_multi_remove_handle($mh, $ch_proj);
curl_multi_close($mh);
```

**Impacto estimado:** o tempo total passa a ser o da chamada mais lenta, não a soma de todas.

---

## 🟡 Médio — Código Morto e Lógica Desnecessária

### Problema
Há blocos inteiros comentados com `<!-- ... -->` e variáveis calculadas (`$agora_completo`, `$p_status_ong_ativo`) que só são usadas dentro desses blocos comentados. O PHP ainda processa as condições, e o HTML ainda carrega o conteúdo comentado.

### Solução
- Remover definitivamente os blocos comentados que não serão reativados.
- Mover o cálculo das variáveis derivadas para dentro dos blocos condicionais onde são realmente usadas.

---

## 🟢 Menor — CSS Inline Repetido

### Problema
O mesmo estilo `box-shadow` é repetido inline em **cada card** de etapa, inflando o HTML desnecessariamente:

```html
<div style="box-shadow: rgba(50, 50, 105, 0.15) 0px 2px 5px 0px, rgba(0, 0, 0, 0.05) 0px 1px 1px 0px;">
```

### Solução
Extrair para uma classe CSS:

```css
.card-etapa {
    box-shadow: rgba(50, 50, 105, 0.15) 0px 2px 5px 0px,
                rgba(0, 0, 0, 0.05) 0px 1px 1px 0px;
}

.card-etapa--concluido {
    background-color: #eeeeee;
}
```

```html
<!-- Antes -->
<div style="background-color:#eeeeee;box-shadow: rgba(50, 50, 105, ...)">

<!-- Depois -->
<div class="card-etapa card-etapa--concluido">
```

---

## Resumo Geral

| # | Melhoria | Prioridade | Impacto | Esforço |
|---|---|---|---|---|
| 2 | Adicionar cache (APCu ou arquivo) | 🟠 Alto | Alto | Médio |
| 3 | Timeout nas chamadas cURL | 🟠 Alto | Médio | Baixo |
| 4 | Paralelizar chamadas com `curl_multi` | 🟡 Médio | Médio | Médio |
| 5 | Remover código morto | 🟡 Médio | Baixo | Baixo |
| 6 | Refatorar CSS inline repetido | 🟢 Menor | Baixo | Baixo |

---
