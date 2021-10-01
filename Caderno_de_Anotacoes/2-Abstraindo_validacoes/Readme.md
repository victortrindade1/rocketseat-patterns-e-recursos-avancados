# Abstraindo validações

<!-- TOC -->

- [Abstraindo validações](#abstraindo-valida%C3%A7%C3%B5es)
  - [src/app/controllers/UserController.js](#srcappcontrollersusercontrollerjs)
  - [src/app/validators/UserStore.js](#srcappvalidatorsuserstorejs)
  - [src/app/validators/UserUpdate.js](#srcappvalidatorsuserupdatejs)
  - [src/app/controllers/SessionController.js](#srcappcontrollerssessioncontrollerjs)
  - [src/app/validators/SessionStore.js](#srcappvalidatorssessionstorejs)
  - [src/app/controllers/AppointmentController.js](#srcappcontrollersappointmentcontrollerjs)
  - [src/app/validators/AppointmentStore.js](#srcappvalidatorsappointmentstorejs)
  - [src/routes.js](#srcroutesjs)

<!-- /TOC -->

Daqui pra frente vamos botar em prática o `pattern Service`. Ou seja, vamos refatorar os controllers. Vou usar o backend gobarber.

Primeira refatoração q farei será tirar as validações dos controllers. Crie a pasta `src/validators`. Migre todas as validações Yup pra validators. Depois chame as validações, mas sem importar pro controller, e sim colocando como middleware nas routes, pq malandro é malandro e mané é mané, diz aí ;)

## src/app/controllers/UserController.js

```diff
-import * as Yup from 'yup';
import User from '../models/User';
import File from '../models/File';

class UserController {
  async store(req, res) {
-    const schema = Yup.object().shape({
-      name: Yup.string().required(),
-      email: Yup.string()
-        .email()
-        .required(),
-      password: Yup.string()
-        .required()
-        .min(6),
-    });
-
-    if (!(await schema.isValid(req.body))) {
-      return res.status(400).json({ error: 'Validation fails' });
-    }
-
    const userExists = await User.findOne({ where: { email: req.body.email } });

    if (userExists) {
      return res.status(400).json({ error: 'User already exists.' });
    }

    // Em vez de carregar na response todos os dados de User, eu escolho carregar estes 4
    const { id, name, email, provider } = await User.create(req.body);

    return res.json({
      id,
      name,
      email,
      provider,
    });
  }

  async update(req, res) {
-    const schema = Yup.object().shape({
-      name: Yup.string(),
-      email: Yup.string().email(),
-      oldPassword: Yup.string().min(6),
-      // Condicional: Se for passado um oldPassword, então o campo password é
-      // obrigatório
-      password: Yup.string()
-        .min(6)
-        .when('oldPassword', (oldPassword, field) =>
-          oldPassword ? field.required() : field
-        ),
-      // Condicional: confirmPassword = password
-      confirmPassword: Yup.string().when('password', (password, field) =>
-        password ? field.required().oneOf([Yup.ref('password')]) : field
-      ),
-    });
-
-    if (!(await schema.isValid(req.body))) {
-      return res.status(400).json({ error: 'Validation fails' });
-    }
-
    const { email, oldPassword } = req.body;

    const user = await User.findByPk(req.userId);

    if (email !== user.email) {
      const userExists = await User.findOne({ where: { email } });

      if (userExists) {
        return res.status(400).json({ error: 'User already exists.' });
      }
    }

    if (oldPassword && !(await user.checkPassword(oldPassword))) {
      return res.status(401).json({ error: 'Password does not match' });
    }

    await user.update(req.body);

    const { id, name, avatar } = await User.findByPk(req.userId, {
      include: [
        {
          model: File,
          as: 'avatar',
          attributes: ['id', 'path', 'url'],
        },
      ],
    });

    return res.json({
      id,
      name,
      email,
      avatar,
    });
  }
}

export default new UserController();
```

## src/app/validators/UserStore.js

> UserStore pq User é do Controller e Store do método do controller

```js
import * as Yup from "yup";

export default async (req, res, next) => {
  try {
    const schema = Yup.object().shape({
      name: Yup.string().required(),
      email: Yup.string().email().required(),
      password: Yup.string().required().min(6),
    });

    await schema.validate(req.body, { abortEarly: false });

    return next();
  } catch (err) {
    return res
      .status(400)
      .json({ error: "Validation fails", messages: err.inner });
  }
};
```

## src/app/validators/UserUpdate.js

```js
import * as Yup from "yup";

export default async (req, res, next) => {
  try {
    const schema = Yup.object().shape({
      name: Yup.string(),
      email: Yup.string().email(),
      oldPassword: Yup.string().min(6),
      // Condicional: Se for passado um oldPassword, então o campo password é
      // obrigatório
      password: Yup.string()
        .min(6)
        .when("oldPassword", (oldPassword, field) =>
          oldPassword ? field.required() : field
        ),
      // Condicional: confirmPassword = password
      confirmPassword: Yup.string().when("password", (password, field) =>
        password ? field.required().oneOf([Yup.ref("password")]) : field
      ),
    });

    await schema.validate(req.body, { abortEarly: false });

    return next();
  } catch (err) {
    return res
      .status(400)
      .json({ error: "Validation fails", messages: err.inner });
  }
};
```

## src/app/controllers/SessionController.js

```diff
-import * as Yup from 'yup';
import jwt from 'jsonwebtoken';
import authConfig from '../../config/auth';
import User from '../models/User';
import File from '../models/File';

class SessionController {
  // Método store -> cria nova session]
  // Repare q store não necessariamente significa gravar algo no banco
  async store(req, res) {
-    const schema = Yup.object().shape({
-      email: Yup.string()
-        .email()
-        .required(),
-      password: Yup.string().required(),
-    });
-
-    if (!(await schema.isValid(req.body))) {
-      return res.status(400).json({ error: 'Validation fails' });
-    }
-
    const { email, password } = req.body;

    const user = await User.findOne({
      where: { email },
      include: [
        {
          model: File,
          as: 'avatar',
          attributes: ['id', 'path', 'url'],
        },
      ],
    });

    if (!user) {
      return res.status(401).json({ error: 'User not found' });
    }

    if (!(await user.checkPassword(password))) {
      return res.status(401).json({ error: 'Password does not match' });
    }

    const { id, name, avatar, provider } = user;

    return res.json({
      user: {
        id,
        name,
        email,
        avatar,
        provider,
      },
      token: jwt.sign({ id }, authConfig.secret, {
        expiresIn: authConfig.expiresIn,
      }),
    });
  }
}

export default new SessionController();
```

## src/app/validators/SessionStore.js

```js
import * as Yup from "yup";

export default async (req, res, next) => {
  try {
    const schema = Yup.object().shape({
      email: Yup.string().email().required(),
      password: Yup.string().required(),
    });

    await schema.validate(req.body, { abortEarly: false });

    return next();
  } catch (err) {
    return res
      .status(400)
      .json({ error: "Validation fails", messages: err.inner });
  }
};
```

## src/app/controllers/AppointmentController.js

```diff
-import * as Yup from 'yup';
import { startOfHour, parseISO, isBefore, format, subHours } from 'date-fns';
import pt from 'date-fns/locale/pt'; // Para traduzir os meses para portugues
import User from '../models/User';
import Appointment from '../models/Appointment';
import File from '../models/File';
import Notification from '../schemas/Notification';

import CancellationMail from '../jobs/CancellationMail';
import Queue from '../../lib/Queue';

class AppointmentController {
  async index(req, res) {
    const { page = 1 } = req.query; // default = página 1

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

    return res.json(appointments);
  }

  async store(req, res) {
-    const schema = Yup.object().shape({
-      provider_id: Yup.number().required(),
-      date: Yup.date().required(),
-    });
-
-    if (!(await schema.isValid(req.body))) {
-      return res.status(400).json({ error: 'Validation fails' });
-    }
-
    const { provider_id, date } = req.body;

    /**
     * Check if provider_id is a provider
     */
    const checkIsProvider = await User.findOne({
      where: { id: provider_id, provider: true },
    });

    if (!checkIsProvider) {
      return res
        .status(401)
        .json({ error: 'You can only create appointments with providers' });
    }

    /**
     * Check if user is not same as the provider
     */
    if (provider_id === req.userId) {
      return res.status(401).json({
        error: 'Providers can not create appointments with themselves',
      });
    }

    // startOfHour é a hora arredondada para baixo
    // parseISO transforma a data num objeto Date
    const hourStart = startOfHour(parseISO(date));

    // Verifica se vai agendar num horário do passado, antes do atual
    if (isBefore(hourStart, new Date())) {
      return res.status(400).json({ error: 'Past dates are not permitted' });
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
      return res
        .status(400)
        .json({ error: 'Appointment date is not available' });
    }

    const appointment = await Appointment.create({
      user_id: req.userId, // userId vem do middleware auth no momento que loga
      provider_id,
      date,
    });

    /**
     * Notificar prestador de serviço
     */
    const user = await User.findByPk(req.userId);
    const formattedDate = format(
      hourStart,
      "'dia' dd 'de' MMMM', às' H:mm'h'",
      { locale: pt }
    );

    await Notification.create({
      content: `Novo agendamento de ${user.name} para ${formattedDate}`,
      user: provider_id,
    });

    return res.json(appointment);
  }

  async delete(req, res) {
    const appointment = await Appointment.findByPk(req.params.id, {
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

    if (appointment.user_id !== req.userId) {
      return res.status(401).json({
        error: "You don't have permission to cancel this appointment.",
      });
    }

    const dateWithSub = subHours(appointment.date, 2); // O campo de data já vem em formato de data. Não precisa de um parseIso pq não é uma string

    // 13:00
    // dateWithSub: 11h
    // now: 11:25h
    // res: horário já passou

    if (isBefore(dateWithSub, new Date())) {
      return res.status(401).json({
        error: 'You can only cancel appointments 2 hours in advance.',
      });
    }

    appointment.canceled_at = new Date();

    await appointment.save();

    await Queue.add(CancellationMail.key, {
      appointment,
      // appointment,
      // foobar: 'foobar',
    });

    return res.json(appointment);
  }
}

export default new AppointmentController();
```

## src/app/validators/AppointmentStore.js

```js
import * as Yup from "yup";

export default async (req, res, next) => {
  try {
    const schema = Yup.object().shape({
      provider_id: Yup.number().required(),
      date: Yup.date().required(),
    });

    await schema.validate(req.body, { abortEarly: false });

    return next();
  } catch (err) {
    return res
      .status(400)
      .json({ error: "Validation fails", messages: err.inner });
  }
};
```

## src/routes.js

```diff
import { Router } from 'express';
import multer from 'multer';
import multerConfig from './config/multer';

import UserController from './app/controllers/UserController';
import SessionController from './app/controllers/SessionController';
import FileController from './app/controllers/FileController';
import ProviderController from './app/controllers/ProviderController';
import AvailableController from './app/controllers/AvailableController';
import AppointmentController from './app/controllers/AppointmentController';
import ScheduleController from './app/controllers/ScheduleController';
import NotificationController from './app/controllers/NotificationController';

+import validateUserStore from './app/validators/UserStore';
+import validateUserUpdate from './app/validators/UserUpdate';
+import validateSessionStore from './app/validators/SessionStore';
+import validateAppointmentStore from './app/validators/AppointmentStore';

import authMiddleware from './app/middlewares/auth';

const routes = new Router();
const upload = multer(multerConfig);

-routes.post('/users', UserController.store);
-routes.post('/sessions', SessionController.store);
+routes.post('/users', validateUserStore, UserController.store);
+routes.post('/sessions', validateSessionStore, SessionController.store);

routes.get('/teste', (req, res) => res.send('ok'));

// Fiz um middleware global para todas as rotas abaixo
// As rotas abaixo são para usuário logado
routes.use(authMiddleware);

// Posso definir authMiddleware de forma local:
// routes.put('/users', authMiddleware, UserController.update);
// Não precisa passar o id pois User possui hash, q lê o id lá no middleware
// auth.js, logo o update fica mais prático neste caso
-routes.put('/users', UserController.update);
+routes.put('/users', validateUserUpdate, UserController.update);

routes.get('/providers', ProviderController.index);
routes.get('/providers/:providerId/available', AvailableController.index);

-routes.post('/appointments', AppointmentController.store);
+routes.post('/appointments', validateAppointmentStore, AppointmentController.store);
routes.get('/appointments', AppointmentController.index);
routes.delete('/appointments/:id', AppointmentController.delete);

routes.get('/schedule', ScheduleController.index);

routes.get('/notifications', NotificationController.index);
routes.put('/notifications/:id', NotificationController.update);

routes.post('/files', upload.single('file'), FileController.store);

export default routes;
```
