# Campaign Management API - Documentação Técnica

## Visão Geral

Esta documentação descreve a API de gerenciamento de ordem de tipos de campanha, incluindo endpoints, regras de negócio e comportamento da interface.

**Base URL:** `https://ipa.infoprice.co/ipa-app/api/ipa/v1`

## Índice

1. [Endpoints](#endpoints)
2. [Regras de Negócio](#regras-de-negócio)
3. [Comportamento da Interface](#comportamento-da-interface)
4. [Exemplos de Implementação](#exemplos-de-implementação)
5. [Tratamento de Erros](#tratamento-de-erros)
6. [Testes](#testes)

## Endpoints

### 1. Tipos de Campanha

#### GET `/campaign/type`
Lista todos os tipos de campanha ordenados por prioridade.

**Resposta (200):**
```json
[
  { "id": "1", "nome": "Anúncio TV", "ordem": 1 },
  { "id": "2", "nome": "Fim de semana", "ordem": 2 },
  { "id": "3", "nome": "Promo quinzenal", "ordem": 3 },
  { "id": "4", "nome": "Gôndola simples", "ordem": 4 }
]
```

#### POST `/campaign/type`
Cria um novo tipo de campanha.

**Request Body:**
```json
{ "name": "XPTO" }
```

**Resposta (201):**
```json
{ "id": "1", "nome": "XPTO", "ordem": 1 }
```

#### PUT `/campaign/type/{id}`
Atualiza o nome de um tipo de campanha.

**Request Body:**
```json
{ "name": "ABCD" }
```

**Resposta (200):**
```json
{ "id": "1", "nome": "ABCD", "ordem": 1 }
```

#### PATCH `/campaign/type-order`
Reordena tipos de campanha trocando posições entre dois itens.

**Request Body:**
```json
{ "firstId": "1", "secondId": "2" }
```

**Resposta (200):**
```json
{ "message": "Campaign types reordered successfully", "success": true }
```

### 2. Configurações de Campanha

#### GET `/setup/campaign`
Obtém configurações globais de campanha.

**Resposta (200):**
```json
{ "priority": "PRIORITY_TYPE", "editOffer": true }
```

#### POST `/setup/campaign`
Atualiza configurações globais de campanha.

**Request Body:**
```json
{ "priority": "PRIORITY_TYPE", "editOffer": true }
```

## Regras de Negócio

### Reordenação de Tipos

1. **Ação de Subir (Move Up):**
   - Enviar ID do item atual + ID do item imediatamente acima
   - Trocar posições dos dois itens

2. **Ação de Descer (Move Down):**
   - Enviar ID do item atual + ID do item imediatamente abaixo
   - Trocar posições dos dois itens

3. **Validações:**
   - Ambos os IDs devem existir
   - IDs devem ser diferentes
   - Operação atômica (falha completa em caso de erro)

### Configurações

- **Priority Types:**
  - `PRIORITY_TYPE`: Priorização por tipo de campanha
  - `PRICE_LOWER`: Priorização por menor preço

- **Edit Offer:**
  - `true`: Permite edição de ofertas
  - `false`: Bloqueia edição de ofertas

## Comportamento da Interface

### Fluxo de Reordenação

```
1. Usuário clica em "subir" ou "descer"
2. Interface exibe loading
3. Envio da requisição PATCH /campaign/type-order
4. Aguarda resposta da API
5. Realiza refetch automático (GET /campaign/type)
6. Atualiza interface com nova ordenação
7. Remove loading
```

### Estados da Interface

- **Loading**: Durante operações de reordenação
- **Disabled**: Botões bloqueados durante operações
- **Error**: Exibição de mensagens de erro
- **Success**: Confirmação visual de sucesso

## Exemplos de Implementação

### JavaScript Vanilla

```javascript
class CampaignTypeManager {
  constructor(apiBaseUrl, authToken) {
    this.apiBaseUrl = apiBaseUrl;
    this.authToken = authToken;
  }

  async getCampaignTypes() {
    const response = await fetch(`${this.apiBaseUrl}/campaign/type`, {
      headers: { 'Authorization': `Bearer ${this.authToken}` }
    });
    return response.json();
  }

  async reorderCampaignTypes(firstId, secondId) {
    const response = await fetch(`${this.apiBaseUrl}/campaign/type-order`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${this.authToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ firstId, secondId })
    });
    
    if (!response.ok) {
      throw new Error(`Reorder failed: ${response.statusText}`);
    }
    
    return response.json();
  }

  async moveUp(currentId, items) {
    const currentIndex = items.findIndex(item => item.id === currentId);
    if (currentIndex > 0) {
      const aboveId = items[currentIndex - 1].id;
      await this.reorderCampaignTypes(currentId, aboveId);
      return this.getCampaignTypes(); // Refetch
    }
  }

  async moveDown(currentId, items) {
    const currentIndex = items.findIndex(item => item.id === currentId);
    if (currentIndex < items.length - 1) {
      const belowId = items[currentIndex + 1].id;
      await this.reorderCampaignTypes(currentId, belowId);
      return this.getCampaignTypes(); // Refetch
    }
  }
}
```

### React Component

```jsx
import React, { useState, useEffect } from 'react';

const CampaignTypeList = () => {
  const [campaignTypes, setCampaignTypes] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const fetchCampaignTypes = async () => {
    try {
      const response = await fetch('/api/ipa/v1/campaign/type');
      const data = await response.json();
      setCampaignTypes(data);
    } catch (err) {
      setError(err.message);
    }
  };

  const handleReorder = async (firstId, secondId) => {
    setLoading(true);
    setError(null);
    
    try {
      await fetch('/api/ipa/v1/campaign/type-order', {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ firstId, secondId })
      });
      
      await fetchCampaignTypes(); // Refetch
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  const moveUp = (index) => {
    if (index > 0) {
      const current = campaignTypes[index];
      const above = campaignTypes[index - 1];
      handleReorder(current.id, above.id);
    }
  };

  const moveDown = (index) => {
    if (index < campaignTypes.length - 1) {
      const current = campaignTypes[index];
      const below = campaignTypes[index + 1];
      handleReorder(current.id, below.id);
    }
  };

  useEffect(() => {
    fetchCampaignTypes();
  }, []);

  return (
    <div>
      {error && <div className="error">{error}</div>}
      {loading && <div className="loading">Reordenando...</div>}
      
      <ul>
        {campaignTypes.map((type, index) => (
          <li key={type.id}>
            <span>{type.nome}</span>
            <div>
              <button 
                onClick={() => moveUp(index)}
                disabled={loading || index === 0}
              >
                ↑ Subir
              </button>
              <button 
                onClick={() => moveDown(index)}
                disabled={loading || index === campaignTypes.length - 1}
              >
                ↓ Descer
              </button>
            </div>
          </li>
        ))}
      </ul>
    </div>
  );
};
```

## Tratamento de Erros

### Códigos de Status

| Código | Descrição | Ação Recomendada |
|--------|-----------|------------------|
| 200/201 | Sucesso | Continuar operação |
| 400 | Requisição inválida | Validar dados enviados |
| 401 | Não autorizado | Renovar token de autenticação |
| 404 | Não encontrado | Verificar se IDs existem |
| 422 | Erro de validação | Corrigir dados conforme retorno |
| 500 | Erro do servidor | Tentar novamente ou reportar |

### Estratégias de Retry

```javascript
const retryOperation = async (operation, maxRetries = 3) => {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (attempt === maxRetries || error.status < 500) {
        throw error;
      }
      
      // Exponential backoff
      await new Promise(resolve => 
        setTimeout(resolve, Math.pow(2, attempt) * 1000)
      );
    }
  }
};
```

## Testes

### Casos de Teste para Reordenação

1. **Teste de Subir Item:**
   - Verificar troca de posições
   - Validar que outros itens não são afetados
   - Confirmar refetch automático

2. **Teste de Descer Item:**
   - Verificar troca de posições
   - Validar que outros itens não são afetados
   - Confirmar refetch automático

3. **Teste de Limites:**
   - Primeiro item não pode subir
   - Último item não pode descer
   - Botões devem estar desabilitados

4. **Teste de Erros:**
   - IDs inexistentes
   - IDs iguais
   - Falha de rede

### Exemplo de Teste (Jest)

```javascript
describe('Campaign Type Reordering', () => {
  test('should move item up successfully', async () => {
    const mockFetch = jest.fn()
      .mockResolvedValueOnce({ ok: true, json: () => ({ success: true }) })
      .mockResolvedValueOnce({ 
        ok: true, 
        json: () => [
          { id: '2', nome: 'Item 2', ordem: 1 },
          { id: '1', nome: 'Item 1', ordem: 2 }
        ]
      });
    
    global.fetch = mockFetch;
    
    const manager = new CampaignTypeManager('http://api.test', 'token');
    const items = [
      { id: '1', nome: 'Item 1', ordem: 1 },
      { id: '2', nome: 'Item 2', ordem: 2 }
    ];
    
    const result = await manager.moveUp('2', items);
    
    expect(mockFetch).toHaveBeenCalledTimes(2);
    expect(result[0].id).toBe('2');
    expect(result[1].id).toBe('1');
  });
});
```

---

**Nota:** Esta documentação deve ser mantida atualizada conforme mudanças na API. Para dúvidas ou sugestões, consulte a equipe de desenvolvimento.
