# Melhorias de Desempenho — Página Home OSC (Engaja Brasil)


## 📊 Tabela de Tempos (Hard Refresh — sem cache)

| Cenário                                              | Tempo (s) |
|-----------------------------------------------------|----------|
| Com links estáticos                                 | 13       |
| Sem links estáticos                                 | 10       |
| Sem API (com links estáticos)                       | 06       |
| Sem API (sem links estáticos)                       | 05       |

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
