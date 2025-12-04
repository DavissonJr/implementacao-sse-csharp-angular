# 📌 Progress Bar em Tempo Real com SSE (Server-Sent Events) – .NET + Angular

Este repositório demonstra como implementar uma **barra de progresso em tempo real** usando **Server-Sent Events (SSE)** entre um backend **.NET 8/9** e um frontend **Angular 17+**.

Ideal para processos longos e divididos em etapas, como:

- Importação de documentos  
- Processamento item a item  
- Etapas internas e sequenciais  
- Execuções demoradas

---

## 🚀 Por que usar SSE?

SSE permite que o servidor envie **atualizações em tempo real** para o navegador sem polling constante.

### Benefícios:
- ✔ Atualizações instantâneas  
- ✔ Baixa complexidade  
- ✔ Perfeito para detalhar etapas de processamento  
- ✔ Sem WebSockets  

---

# 🛠 Tecnologias

- Backend: **.NET 8/9 (Minimal API)**  
- Frontend: **Angular 17+**  
- Comunicação: **SSE (EventSource)**  

---

# -----------------------------------------
# 🧩 BACKEND (.NET) – IMPLEMENTAÇÃO SSE
# -----------------------------------------

## 1. Registrar os serviços no Program.cs

```csharp
builder.Services.AddSingleton<ProgressService>();
builder.Services.AddSingleton<ImportacaoService>();
```
## 2. Endpoint SSE para enviar atualizações em tempo real

```csharp
app.MapGet("/importar/sse/{processId}", async (string processId, HttpContext context) =>
{
    context.Response.Headers.Append("Content-Type", "text/event-stream");
    context.Response.Headers.Append("Cache-Control", "no-cache");
    context.Response.Headers.Append("Connection", "keep-alive");

    var progressService = context.RequestServices.GetRequiredService<ProgressService>();

    await foreach (var update in progressService.StreamUpdates(processId))
    {
        await context.Response.WriteAsync($"data: {update}\n\n");
        await context.Response.Body.FlushAsync();
    }
});
```

## 3. Serviço que gerencia o envio de mensagens SSE

```csharp
public class ProgressService
{
    private readonly Dictionary<string, Channel<string>> _channels = new();

    public Channel<string> GetChannel(string processId)
    {
        if (!_channels.ContainsKey(processId))
            _channels[processId] = Channel.CreateUnbounded<string>();

        return _channels[processId];
    }

    public async Task SendUpdate(string processId, object data)
    {
        var channel = GetChannel(processId);
        await channel.Writer.WriteAsync(JsonSerializer.Serialize(data));
    }

    public async IAsyncEnumerable<string> StreamUpdates(string processId)
    {
        var channel = GetChannel(processId);

        while (await channel.Reader.WaitToReadAsync())
        {
            while (channel.Reader.TryRead(out var msg))
                yield return msg;
        }
    }
}
```
## 4. Serviço responsável pelo processo demorado
```csharp
public class ImportacaoService
{
    private readonly ProgressService _progress;

    public ImportacaoService(ProgressService progress)
    {
        _progress = progress;
    }

    public async Task ProcessarAsync(string processId)
    {
        await _progress.SendUpdate(processId, new { etapa = "Separando documentos", progress = 10 });
        await Task.Delay(1500);

        await _progress.SendUpdate(processId, new { etapa = "Validando arquivos", progress = 35 });
        await Task.Delay(1500);

        await _progress.SendUpdate(processId, new { etapa = "Convertendo arquivos", progress = 65 });
        await Task.Delay(1500);

        await _progress.SendUpdate(processId, new { etapa = "Finalizando", progress = 100 });
    }
}
```
## 5. Endpoint para iniciar o processo

```csharp
app.MapPost("/importar", async (ImportacaoService import) =>
{
    var processId = Guid.NewGuid().ToString();

    _ = Task.Run(() => import.ProcessarAsync(processId));

    return Results.Ok(new { processId });
});
```

# -----------------------------------------
# 🌐 FRONTEND (ANGULAR) – IMPLEMENTAÇÃO SSE
# -----------------------------------------

## 1. Criar o service para consumir o SSE

Crie o arquivo:

`src/app/services/progress.service.ts`

```ts
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root',
})
export class ProgressService {
  connect(processId: string): Observable<any> {
    return new Observable(observer => {
      const eventSource = new EventSource(`http://localhost:5000/importar/sse/${processId}`);

      eventSource.onmessage = event => {
        observer.next(JSON.parse(event.data));
      };

      eventSource.onerror = error => {
        observer.error(error);
        eventSource.close();
      };

      return () => eventSource.close();
    });
  }
}
```

## 2. Criar o componente que consome o SSE

`src/app/importar/importar.component.ts`

```ts
import { Component } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { ProgressService } from '../services/progress.service';

@Component({
  selector: 'app-importar',
  templateUrl: './importar.component.html',
  styleUrls: ['./importar.component.css']
})
export class ImportarComponent {
  etapa = '';
  progress = 0;

  constructor(
    private http: HttpClient,
    private progressService: ProgressService
  ) {}

  iniciar() {
    this.http.post<any>('http://localhost:5000/importar', {})
      .subscribe(res => {
        const processId = res.processId;

        this.progressService.connect(processId).subscribe(update => {
          this.etapa = update.etapa;
          this.progress = update.progress;
        });
      });
  }
}
```

## 3. Criar o HTML com a barra de progresso

`src/app/importar/importar.component.html`

```html
<button (click)="iniciar()">Iniciar Importação</button>

<div *ngIf="progress > 0">
  <p><strong>Etapa:</strong> {{ etapa }}</p>

  <div class="progress-container">
    <div class="progress-bar" [style.width.%]="progress"></div>
  </div>

  <p>{{ progress }}%</p>
</div>
```
## 4. Estilos da barra de progresso

`src/app/importar/importar.component.css`
