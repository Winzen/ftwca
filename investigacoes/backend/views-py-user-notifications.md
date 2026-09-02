```py
# -*- coding: utf-8 -*-

# ruff: noqa: E501, F841

# -*- coding: utf-8 -*-

# Create your views here.

from uuid import UUID

  

from django.http import JsonResponse

from django.utils import timezone

from django.views.decorators.csrf import csrf_exempt

from django.contrib.auth.decorators import login_required

  

from .models import Account, Table, TableUpdateSubscription

from backend.apps.account.token import token_generator

  

@csrf_exempt

def add_subscription(request):

    if request.method == "POST":

        table_id = request.POST.get("table_id")

        user_id = request.POST.get("user_id")

  

        try:

            table = Table.objects.get(id=UUID(table_id))

            user = Account.objects.get(id=user_id)

  

            # Verificando se já existe uma assinatura ativa para esse usuário e essa tabela

            existing_subscription = TableUpdateSubscription.objects.filter(

                table=table, user=user, status=True

            ).first()

  

            if existing_subscription:

                # Se já existe uma assinatura ativa, retornamos um erro

                return JsonResponse(

                    {

                        "status": "error",

                        "code": "Já existe uma assinatura ativa",

                        "message": f"Já existe uma assinatura ativa para a tabela {table.name} e o usuário {user.username}.",

                    }

                )

  

            # Criando a nova assinatura

            TableUpdateSubscription.objects.create(table=table, user=user, status=True)

  

            # Respondendo com sucesso

            return JsonResponse(

                {

                    "status": "success",

                    "message": f"Assinatura criada para a tabela {table.name} e usuário {user.username}.",

                }

            )

  

        except Table.DoesNotExist:

            return JsonResponse(

                {

                    "status": "error",

                    "code": "Tabela não encontrada.",

                    "message": "Tabela não encontrada.",

                }

            )

        except Account.DoesNotExist:

            return JsonResponse(

                {

                    "status": "error",

                    "code": "Usuário não encontrado.",

                    "message": "Usuário não encontrado.",

                }

            )

    else:

        return JsonResponse(

            {

                "status": "error",

                "code": "Método de requisição inválido.",

                "message": "Método de requisição inválido.",

            }

        )

  

@csrf_exempt

def remove_subscription(request):

    if request.method == "POST":

        table_id = request.POST.get("table_id")

        user_id = request.POST.get("user_id")

  

        try:

            # Buscando a tabela e o usuário

            table = Table.objects.get(id=UUID(table_id))

            user = Account.objects.get(id=user_id)

  

            # Verificando se existe uma assinatura ativa para esse usuário e essa tabela

            subscription = TableUpdateSubscription.objects.filter(

                table=table,

                user=user,

                status=True,  # Apenas as assinaturas ativas

            ).first()

            if not subscription:

                # Se não houver assinatura ativa, retornamos um erro

                return JsonResponse(

                    {

                        "status": "error",

                        "code": "Não existe uma assinatura ativa",

                        "message": f"Não existe uma assinatura ativa para a tabela {table.name} e o usuário {user.username}.",

                    }

                )

  

            # Atualizando o status para False e registrando a data de desativação

            subscription.status = False

            subscription.deleted_at = timezone.now()  # Atualizando com a data e hora atual

            subscription.save()

  

            # Respondendo com sucesso

            return JsonResponse(

                {

                    "status": "success",

                    "message": f"Assinatura desativada para a tabela {table.name} e o usuário {user.username}.",

                }

            )

  

        except Table.DoesNotExist:

            return JsonResponse(

                {

                    "status": "error",

                    "code": "Tabela não encontrada.",

                    "message": "Tabela não encontrada.",

                }

            )

        except Account.DoesNotExist:

            return JsonResponse(

                {

                    "status": "error",

                    "code": "Usuário não encontrado.",

                    "message": "Usuário não encontrado.",

                }

            )

    else:

        return JsonResponse(

            {

                "status": "error",

                "code": "Método de requisição inválido.",

                "message": "Método de requisição inválido.",

            }

        )
```