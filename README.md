# Fichamento NFLua

## O que é o NFLua?

O NFLua é um binding para o NetFilter, que por sua vez é o firewall do linux. O NFLua permite o registro e a execução de código em lua para os hooks das operações do NetFilter.

## Operações Oferecidas pelo NFLua

- `struct nflua_state *nflua_state_create(struct xt_lua_net *xt_lua,
	size_t maxalloc, const char *name)` para a criação de estados.
- `int nflua_state_destroy(struct xt_lua_net *xt_lua, const char *name)` para a delação de estados.
- `void nflua_states_init(struct xt_lua_net *xt_lua)` para a iniciazliação dos estados (inicializar os spinslocks da estrutura que engloba os estados e inicializar a cabeça de cada um dos buckets utilizados para armazenar os estados).
- `void nflua_state_destroy_all(struct xt_lua_net *xt_lua)` destroi todos os estados presente em um namespace.
- `void nflua_states_exit(struct xt_lua_net *xt_lua)` sai (para de utilizar) do nflua, somente faz uma chamada à função `nflua_state_destroy_all`.
- `int nflua_state_list(struct xt_lua_net *xt_lua,nflua_state_cb cb,unsigned short *total)` utilizada para listar os estados presentes em um namespace.
- `struct nflua_state *nflua_state_lookup(struct xt_lua_net *xt_lua,const char *name)` utilizada para verificar a existência de um determinado estado.

Todas as funções citadas acima estão implementadas no arquivo `src/states.c` e oferecem uma API para a manipulação de estados dentro do ambiente do NFLua.


## A `struct nflua_state`

Definida como: 

```
struct nflua_state {
	struct hlist_node node;
	lua_State *L;
	struct xt_lua_net *xt_lua;
	spinlock_t lock;
	kpi_refcount_t users;
	u32 dseqnum;
	size_t maxalloc;
	size_t curralloc;
	unsigned char name[NFLUA_NAME_MAXSIZE];
};
```

- `node` utilizado para permitir que a `struct nflua_state` seja armazenada em uma hash table.
- `L` estado lua utilizado para execução de código, armazenamento do contexto etc.
- `xt_lua` referência ao namespace à que a `struct nflua_state` se refere.
- `lock` variável utilizada para a sincronização entre os usuários da struct.
<span style="color:blue"> - `users` apenas um teste</span> 
