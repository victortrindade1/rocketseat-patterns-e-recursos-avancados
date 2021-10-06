# Abstraindo agendamento

Nem tudo sai dos controllers. Se ele apenas faz uma querie no banco, beleza, a gnt larga lá. Agora, se dessa querie, ele tem q tratar os valores da response, usar lógicas, aí a gnt separa pra uma pasta `src/service`.

O ideal é ter **um service pra cada lógica** isolada do app. Ou seja, se um controller tem mais de um método, crie um service pra cada método separado. Conforme o app cresce, mais vezes serão necessários chamar estes services pra outras features do app. E se vc faz um service pro controller inteiro, então vc está trocando 6 por meia-dúzia, pq teu service vai continuar gigante assim como o controller era.

> Fique ligado. Não é pros services manipularem os objetos req e res. Estes, são passados como parâmetro. A lógica do service fica toda isolada!

## src/app/controllers/AppointmentController.js

```diff
-import { startOfHour, parseISO, isBefore, format, subHours } from 'date-fns';
+import { isBefore, subHours } from 'date-fns';
-import pt from 'date-fns/locale/pt'; // Para traduzir os meses para portugues
import User from '../models/User';
import Appointment from '../models/Appointment';
import File from '../models/File';
-import Notification from '../schemas/Notification';

import CancellationMail from '../jobs/CancellationMail';
import Queue from '../../lib/Queue';

+import CreateAppointmentService from '../services/CreateAppointmentService';

class AppointmentController {
  async index(req, res) {
    ...
  }

  async store(req, res) {
    const { provider_id, date } = req.body;

-    /**
-     * Check if provider_id is a provider
-     */
-    const checkIsProvider = await User.findOne({
-      where: { id: provider_id, provider: true },
-    });
-
-    if (!checkIsProvider) {
-      return res
-        .status(401)
-        .json({ error: 'You can only create appointments with providers' });
-    }
-
-    /**
-     * Check if user is not same as the provider
-     */
-    if (provider_id === req.userId) {
-      return res.status(401).json({
-        error: 'Providers can not create appointments with themselves',
-      });
-    }
-
-    // startOfHour é a hora arredondada para baixo
-    // parseISO transforma a data num objeto Date
-    const hourStart = startOfHour(parseISO(date));
-
-    // Verifica se vai agendar num horário do passado, antes do atual
-    if (isBefore(hourStart, new Date())) {
-      return res.status(400).json({ error: 'Past dates are not permitted' });
-    }
-
-    /**
-     * Check date availability
-     */
-    const checkAvailability = await Appointment.findOne({
-      where: {
-        provider_id,
-        canceled_at: null,
-        date: hourStart,
-      },
-    });
-
-    if (checkAvailability) {
-      return res
-        .status(400)
-        .json({ error: 'Appointment date is not available' });
-    }
-
-    const appointment = await Appointment.create({
-      user_id: req.userId, // userId vem do middleware auth no momento que loga
-      provider_id,
-      date,
-    });
-
-    /**
-     * Notificar prestador de serviço
-     */
-    const user = await User.findByPk(req.userId);
-    const formattedDate = format(
-      hourStart,
-      "'dia' dd 'de' MMMM', às' H:mm'h'",
-      { locale: pt }
-    );
-
-    await Notification.create({
-      content: `Novo agendamento de ${user.name} para ${formattedDate}`,
-      user: provider_id,
-    });
+    const appointment = await CreateAppointmentService.run({
+      provider_id,
+      user_id: req.userId,
+      date,
+    });

    return res.json(appointment);
  }

  async delete(req, res) {
    ...
  }
}

export default new AppointmentController();
```

## src/app/services/CreateAppointmentService.js

- Services precisam estar isolados de `req` e `res`. Vamos eliminá-los então!
- A informação de req eu passei sendo um objeto como parâmetro (`{ user_id: req.userId }`). Não precisaria ser um objeto. O bom de ser, é q dou nomes às variáveis. Assim, fica limpo passar user_id apenas.
- Onde tinha `res.status(400)`, agora vai dar um simples `throw new Error()`, q possui o mesmo efeito
- Acrescentou um return. Todo service retorna alguma coisa.

```js
import { startOfHour, parseISO, isBefore, format } from "date-fns";
import pt from "date-fns/locale/pt"; // Para traduzir os meses para portugues

import User from "../models/User";
import Appointment from "../models/Appointment";

import Notification from "../schemas/Notification";

class CreateAppointmentService {
  async run({ provider_id, user_id, date }) {
    /**
     * Check if provider_id is a provider
     */
    const checkIsProvider = await User.findOne({
      where: { id: provider_id, provider: true },
    });

    if (!checkIsProvider) {
      throw new Error("You can only create appointments with providers");
    }

    /**
     * Check if user is not same as the provider
     */
    if (provider_id === user_id) {
      throw new Error("Providers can not create appointments with themselves");
    }

    // startOfHour é a hora arredondada para baixo
    // parseISO transforma a data num objeto Date
    const hourStart = startOfHour(parseISO(date));

    // Verifica se vai agendar num horário do passado, antes do atual
    if (isBefore(hourStart, new Date())) {
      throw new Error("Past dates are not permitted");
    }

    /**
     * Check date availability
     */
    const checkAvailability = await Appointment.findOne({
      where: {
        provider_id,
        canceled_at: null,
        date: hourStart,
      },
    });

    if (checkAvailability) {
      throw new Error("Appointment date is not available");
    }

    const appointment = await Appointment.create({
      user_id, // userId vem do middleware auth no momento que loga
      provider_id,
      date,
    });

    /**
     * Notificar prestador de serviço
     */
    const user = await User.findByPk(user_id);
    const formattedDate = format(
      hourStart,
      "'dia' dd 'de' MMMM', às' H:mm'h'",
      { locale: pt }
    );

    await Notification.create({
      content: `Novo agendamento de ${user.name} para ${formattedDate}`,
      user: provider_id,
    });

    return appointment;
  }
}

export default new CreateAppointmentService();
```
