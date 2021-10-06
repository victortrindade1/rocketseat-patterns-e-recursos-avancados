# Abstraindo controllers

## src/app/controllers/AppointmentController.js

```diff
-import { isBefore, subHours } from 'date-fns';
import User from '../models/User';
import Appointment from '../models/Appointment';
import File from '../models/File';

-import CancellationMail from '../jobs/CancellationMail';
-import Queue from '../../lib/Queue';

import CreateAppointmentService from '../services/CreateAppointmentService';
+import CancelAppointmentService from '../services/CancelAppointmentService';

class AppointmentController {
  async index(req, res) {
    ...
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
-    const appointment = await Appointment.findByPk(req.params.id, {
-      include: [
-        {
-          model: User,
-          as: 'provider',
-          attributes: ['name', 'email'],
-        },
-        {
-          model: User,
-          as: 'user',
-          attributes: ['name'],
-        },
-      ],
-    });
-
-    if (appointment.user_id !== req.userId) {
-      return res.status(401).json({
-        error: "You don't have permission to cancel this appointment.",
-      });
-    }
-
-    const dateWithSub = subHours(appointment.date, 2); // O campo de data já vem em formato de data. Não precisa de um parseIso pq não é uma string

-    // 13:00
-    // dateWithSub: 11h
-    // now: 11:25h
-    // res: horário já passou

-    if (isBefore(dateWithSub, new Date())) {
-      return res.status(401).json({
-        error: 'You can only cancel appointments 2 hours in advance.',
-      });
-    }
-
-    appointment.canceled_at = new Date();
-
-    await appointment.save();
-
-    await Queue.add(CancellationMail.key, {
-      appointment,
-      // appointment,
-      // foobar: 'foobar',
-    });
+    const appointment = await CancelAppointmentService.run({
+      provider_id: req.params.id,
+      user_id: req.userId,
+    });

    return res.json(appointment);
  }
}

export default new AppointmentController();
```

## src/app/services/CancelAppointmentService.js

```js
import { isBefore, subHours } from "date-fns";

import Appointment from "../models/Appointment";
import User from "../models/User";

import CancellationMail from "../jobs/CancellationMail";
import Queue from "../../lib/Queue";

class CancelAppointmentService {
  async run({ provider_id, user_id }) {
    const appointment = await Appointment.findByPk(provider_id, {
      include: [
        {
          model: User,
          as: "provider",
          attributes: ["name", "email"],
        },
        {
          model: User,
          as: "user",
          attributes: ["name"],
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
      throw new Error("You can only cancel appointments 2 hours in advance.");
    }

    appointment.canceled_at = new Date();

    await appointment.save();

    await Queue.add(CancellationMail.key, {
      appointment,
      // appointment,
      // foobar: 'foobar',
    });

    return appointment;
  }
}

export default new CancelAppointmentService();
```

## src/app/controllers/AvailableController.js

```diff
-import {
-  startOfDay,
-  endOfDay,
-  setHours,
-  setMinutes,
-  setSeconds,
-  format,
-  isAfter,
-} from 'date-fns';
-import { Op } from 'sequelize';
-import Appointment from '../models/Appointment';
+import AvailableService from '../services/AvailableService';

class AvailableController {
  async index(req, res) {
    const { date } = req.query;

    if (!date) {
      return res.status(400).json({ error: 'Invalid date' });
    }

    const searchDate = Number(date);

-    const appointments = await Appointment.findAll({
-      where: {
-        provider_id: req.params.providerId,
-        canceled_at: null,
-        date: {
-          [Op.between]: [startOfDay(searchDate), endOfDay(searchDate)],
-        },
-      },
-    });
-
-    const schedule = [
-      '08:00',
-      '09:00',
-      '10:00',
-      '11:00',
-      '12:00',
-      '13:00',
-      '14:00',
-      '15:00',
-      '16:00',
-      '17:00',
-      '18:00',
-    ];
-
-    const available = schedule.map(time => {
-      const [hour, minute] = time.split(':');
-      const value = setSeconds(
-        setMinutes(setHours(searchDate, hour), minute),
-        0
-      );
-
-      return {
-        time,
-        value: format(value, "yyyy-MM-dd'T'HH:mm:ssxxx"),
-        available:
-          isAfter(value, new Date()) &&
-          !appointments.find(a => format(a.date, 'HH:mm') === time),
-      };
-    });
+    const available = await AvailableService.run({
+      date: searchDate,
+      provider_id: req.params.providerId,
+    });

    return res.json(available);
  }
}

export default new AvailableController();
```

## src/app/services/AvailableService.js

```js
import {
  startOfDay,
  endOfDay,
  setHours,
  setMinutes,
  setSeconds,
  format,
  isAfter,
} from "date-fns";
import { Op } from "sequelize";

import Appointment from "../models/Appointment";

class AvailableService {
  async run({ date, provider_id }) {
    const appointments = await Appointment.findAll({
      where: {
        provider_id,
        canceled_at: null,
        date: {
          [Op.between]: [startOfDay(date), endOfDay(date)],
        },
      },
    });

    const schedule = [
      "08:00",
      "09:00",
      "10:00",
      "11:00",
      "12:00",
      "13:00",
      "14:00",
      "15:00",
      "16:00",
      "17:00",
      "18:00",
    ];

    const available = schedule.map((time) => {
      const [hour, minute] = time.split(":");
      const value = setSeconds(setMinutes(setHours(date, hour), minute), 0);

      return {
        time,
        value: format(value, "yyyy-MM-dd'T'HH:mm:ssxxx"),
        available:
          isAfter(value, new Date()) &&
          !appointments.find((a) => format(a.date, "HH:mm") === time),
      };
    });

    return available;
  }
}

export default new AvailableService();
```
