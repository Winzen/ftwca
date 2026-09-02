
Estou fazendo um comando no django que pega uma modelo que tem uma lista de interesses de usuários por diferentes tabelas segue o modelo:

class TableUpdateSubscription(BaseModel):

table = models.ForeignKey(Table, on_delete=models.CASCADE)

user = models.ForeignKey(Account, on_delete=models.CASCADE)

created_at = models.DateTimeField(auto_now_add=True)

deactivate_at = models.DateTimeField(null=True, blank=True)

last_notification = models.DateTimeField(null=True, blank=True)

updated_at = models.DateTimeField(null=True, blank=True)

status = models.BooleanField(default=True) # True = Ativa, False = Inativa

def __str__(self):

return f"Subscription {self.id} - Table: {self.table.name}, User: {self.user.username}"

class Meta:

verbose_name = "Table Update Subscription"

verbose_name_plural = "Table Update Subscriptions"

Preciso pegar ela, vê interesses com status ativo, pega o campo updated_at e comparar com a tabela de Table que tem um campo chamado "last_updated_at" para ver quais interesses estão desatualizados para mandar emails avisando que tem atualizações disponíveis para as tabelas que eles tem interesse


### Templete email de notificação para mudar a tag a

```
<a href="#" target="_blank" style="text-decoration:none; font-weight:700; color:#000000;">

  <strong style="color:#000000;">Tabela:</strong> 
  <span style="color:#2b8c4d;">Microdados 2019</span>
  &nbsp; ( 
  <strong style="color:#000000;">Dataset:</strong> 
  <span style="color:#2b8c4d;">Pesquisa Nacional de Saúde (PNS)</span> )

</a>
```

