# Estratégias de cache

Vou criar o cache pra listagem de agendamentos. Agora é um pouco mais complexo, pq o resultado da lista difere pra cada usuário, e pra cada página.

## src/lib/Cache.js

Vou ter q adicionar um método pra deletar várias chaves de uma vez de um mesmo usuário. Isto pq o redis vai agora cadastrar as consultas por usuario por página.

```diff
import Redis from 'ioredis';

class Cache {
  constructor() {
    this.redis = new Redis({
      host: process.env.REDIS_HOST,
      port: process.env.REDIS_PORT,
      keyPrefix: 'cache:',
    });
  }

  set(key, value) {
    // Deleta cache depois de 24h (60s * 60min * 24h)
    // EX = excluir
    return this.redis.set(key, JSON.stringify(value), 'EX', 60 * 60 * 24);
  }

  async get(key) {
    const cached = await this.redis.get(key);

    return cached ? JSON.parse(cached) : null;
  }

  invalidate(key) {
    return this.redis.del(key);
  }

+  async invalidatePrefix(prefix) {
+    const keys = await this.redis.keys(`cache:${prefix}:*`);
+
+    const keysWithoutPrefix = keys.map(key => key.replace('cache:', ''));
+
+    return this.redis.del(keysWithoutPrefix);
+  }
}

export default new Cache();
```

## src/app/controllers/AppointmentController.js

```diff
import User from '../models/User';
import Appointment from '../models/Appointment';
import File from '../models/File';

import CreateAppointmentService from '../services/CreateAppointmentService';
import CancelAppointmentService from '../services/CancelAppointmentService';

+import Cache from '../../lib/Cache';

class AppointmentController {
  async index(req, res) {
    const { page = 1 } = req.query; // default = página 1

+    const cacheKey = `user:${req.userId}:appointments:${page}`;
+    const cached = await Cache.get(cacheKey);
+
+    if (cached) {
+      return res.json(cached);
+    }

    const appointments = await Appointment.findAll({
      where: { user_id: req.userId, canceled_at: null },
      order: ['date'],
      attributes: ['id', 'date', 'past', 'cancelable'],
      limit: 20, // mostra até 20 agendamentos por página
      offset: (page - 1) * 20,
      include: [
        {
          model: User,
          as: 'provider',
          attributes: ['id', 'name'],
          include: [
            {
              model: File,
              as: 'avatar',
              // Se não colocar id como atributo, a url tb não aparece
              // Se não colocar path, aparece undefined, pq path é variável da url lá no model File
              attributes: ['id', 'path', 'url'],
            },
          ],
        },
      ],
    });

+    await Cache.set(cacheKey, appointments);

    return res.json(appointments);
  }

  async store(req, res) {
    const { provider_id, date } = req.body;

    const appointment = await CreateAppointmentService.run({
      provider_id,
      user_id: req.userId,
      date,
    });

    return res.json(appointment);
  }

  async delete(req, res) {
    const appointment = await CancelAppointmentService.run({
      provider_id: req.params.id,
      user_id: req.userId,
    });

    return res.json(appointment);
  }
}

export default new AppointmentController();
```

## src/app/services/CreateAppointmentService.js

```diff
import { startOfHour, parseISO, isBefore, format } from 'date-fns';
import pt from 'date-fns/locale/pt'; // Para traduzir os meses para portugues

import User from '../models/User';
import Appointment from '../models/Appointment';

import Notification from '../schemas/Notification';

+import Cache from '../../lib/Cache';

class CreateAppointmentService {
  async run({ provider_id, user_id, date }) {
    /**
     * Check if provider_id is a provider
     */
    const checkIsProvider = await User.findOne({
      where: { id: provider_id, provider: true },
    });

    if (!checkIsProvider) {
      throw new Error('You can only create appointments with providers');
    }

    /**
     * Check if user is not same as the provider
     */
    if (provider_id === user_id) {
      throw new Error('Providers can not create appointments with themselves');
    }

    // startOfHour é a hora arredondada para baixo
    // parseISO transforma a data num objeto Date
    const hourStart = startOfHour(parseISO(date));

    // Verifica se vai agendar num horário do passado, antes do atual
    if (isBefore(hourStart, new Date())) {
      throw new Error('Past dates are not permitted');
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
      throw new Error('Appointment date is not available');
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

+    /**
+     * Invalidate cache
+     */
+    await Cache.invalidatePrefix(`user:${user_id}:appointments`);

    return appointment;
  }
}

export default new CreateAppointmentService();
```

## src/app/services/CancelAppointmentService.js

```diff
import { isBefore, subHours } from 'date-fns';

import Appointment from '../models/Appointment';
import User from '../models/User';

import CancellationMail from '../jobs/CancellationMail';

import Queue from '../../lib/Queue';
+import Cache from '../../lib/Cache';

class CancelAppointmentService {
  async run({ provider_id, user_id }) {
    const appointment = await Appointment.findByPk(provider_id, {
      include: [
        {
          model: User,
          as: 'provider',
          attributes: ['name', 'email'],
        },
        {
          model: User,
          as: 'user',
          attributes: ['name'],
        },
      ],
    });

    if (appointment.user_id !== user_id) {
      throw new Error("You don't have permission to cancel this appointment.");
    }

    const dateWithSub = subHours(appointment.date, 2); // O campo de data já vem em formato de data. Não precisa de um parseIso pq não é uma string

    // 13:00
    // dateWithSub: 11h
    // now: 11:25h
    // res: horário já passou

    if (isBefore(dateWithSub, new Date())) {
      throw new Error('You can only cancel appointments 2 hours in advance.');
    }

    appointment.canceled_at = new Date();

    await appointment.save();

    await Queue.add(CancellationMail.key, {
      appointment,
      // appointment,
      // foobar: 'foobar',
    });

+    /**
+     * Invalidate cache
+     */
+    await Cache.invalidatePrefix(`user:${user_id}:appointments`);

    return appointment;
  }
}

export default new CancelAppointmentService();
```
