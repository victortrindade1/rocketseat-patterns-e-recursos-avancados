# Cache de prestadores

Vamos configurar um cache manualmente. O usuário quer listar todos os prestadores de serviço. Vou fazer um cache pra essa consulta. O cache será invalidado quando um novo prestador de serviço for cadastrado. Assim, mesmo com o cache, os resultados da tela são sempre atuais.

> Não é pra sair tacando cache em todas as queries de db. Coloque cache nas consultas mais pesadas, com maior número de execuções iguais sem atualizações. Ex: carregar tela de dashboard é uma boa.

## src/app/controllers/ProviderController.js

```diff
import User from '../models/User';
import File from '../models/File';

+import Cache from '../../lib/Cache';

class ProviderController {
  async index(req, res) {
+    const cached = await Cache.get('providers');
+
+    if (cached) {
+      return res.json(cached);
+    }

    const providers = await User.findAll({
      where: { provider: true },
      // Os campos q eu quero q mostre ficam em "attributes"
      attributes: ['id', 'name', 'email', 'avatar_id'],
      include: [
        {
          model: File,
          as: 'avatar',
          attributes: ['name', 'path', 'url'],
        },
      ],
    });

+    await Cache.set('providers', providers);

    return res.json(providers);
  }
}

export default new ProviderController();
```

## src/app/controllers/UserController.js

```diff
import User from '../models/User';
import File from '../models/File';

+import Cache from '../../lib/Cache';

class UserController {
  async store(req, res) {
    const userExists = await User.findOne({ where: { email: req.body.email } });

    if (userExists) {
      return res.status(400).json({ error: 'User already exists.' });
    }

    // Em vez de carregar na response todos os dados de User, eu escolho carregar estes 4
    const { id, name, email, provider } = await User.create(req.body);

+    if (provider) {
+      await Cache.invalidate('providers');
+    }

    return res.json({
      id,
      name,
      email,
      provider,
    });
  }

  async update(req, res) {
    ...
  }
}

export default new UserController();
```
