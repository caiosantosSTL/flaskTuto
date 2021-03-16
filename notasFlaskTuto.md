# Notas flask tuto

Iniciamos criando um arquivo \_init_ no qual vamos colocar as configuracoes principiais do sistema.

Criamos um arquivo sql para a criacao de nosso banco de dados sql.

Criamos um arquivo python para o banco de dados, que vai permitir criar o arquivo sqlite em uma pasta especifica. E definimos o comando init-db que usaremos para criar o db.

## Auth

Temos o arquivo python de autentificacao, onde:

```py
import functools

from flask import (
    Blueprint, flash, g, redirect, render_template, request, session, url_for
)
from werkzeug.security import check_password_hash, generate_password_hash

from projtuto.db import get_db
```

> Importamos esses valores

Vemos no metodo de registro:

### Registro

```py
@bp.route('/register', methods=('GET', 'POST'))
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        db = get_db()
        error = None
```

Indo para a tal rota, entramos no controlador no qual, se o request for de tipo post, refecendo do formulario o nome e senha, pegamos as informacoes do db e colocamos erro como nenhum.

Em seguida, criamos as condicoes ao colocar as informacoes no formulario:

```py
        if not username:
            error = 'Username is required.'
        elif not password:
            error = 'Password is required.'
        elif db.execute(
            'SELECT id FROM user WHERE username = ?', (username,)
        ).fetchone() is not None:
            error = 'User {} is already registered.'.format(username)
```

Se o nome e senha tiverem caracteristicas nao aceitaveis, vai nos dar um erro e sua mensagem, caso as informacoes colocadas forem boas, executamos o db para fazer um select e achar se existe o nome repetido no db, caso exista, dara uma mensagem que ja foi registrado tal nome.

Em seguida temos:

```py
        if error is None:
            db.execute(
                'INSERT INTO user (username, password) VALUES (?, ?)',
                (username, generate_password_hash(password))
            )
            db.commit()
            return redirect(url_for('auth.login'))

        flash(error)

    return render_template('auth/register.html')
```

Se nao tiver erro do nome repetido, fazemos um insert no db, ao fazer um commit no db, fazemos um retorno para a pagina de login. No final do codigo, serve para retornar a tela de registro.

### Login

```py
@bp.route('/login', methods=('GET', 'POST'))
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        db = get_db()
        error = None
        user = db.execute(
            'SELECT * FROM user WHERE username = ?', (username,)
        ).fetchone()

        if user is None:
            error = 'Incorrect username.'
        elif not check_password_hash(user['password'], password):
            error = 'Incorrect password.'
```

Esta parte do codigo serve para indicar se tais informacoes colocadas, existem ou nao no db, por meio de um select.

Em seguida temos:

```py
        if error is None:
            session.clear()
            session['user_id'] = user['id']
            return redirect(url_for('index'))

        flash(error)

    return render_template('auth/login.html')
```

Se nao tiver nenhum erro no registro, limpamos uma sessao se tiver, e usaamos o id do usuario como elemento de sessao, e retornamos para a tela de index, onde podemos fazer os posts.

```py
# area request
@bp.before_app_request
def load_logged_in_user():
    user_id = session.get('user_id')

    if user_id is None:
        g.user = None
    else:
        g.user = get_db().execute(
            'SELECT * FROM user WHERE id = ?', (user_id,)
        ).fetchone()
```

Aqui fazemos outra validacao, onde pegamos o id do usuario e vemos se tal existe no db ou nao.

### Sair / terminar sessao

```py
#rota logout / terminar sessao
@bp.route('/logout')
def logout():
    session.clear()
    return redirect(url_for('index'))
```

Aqui serve para quando apertar o botao de sair, entramos em tal rota e executamos esse controlador no qual limpa a sessao e redireciona para a pagina de index.

## Blog

Aqui temos a descriacao do codigo do arquivo blog python.

```py
from flask import (
    Blueprint, flash, g, redirect, render_template, request, url_for
)
from werkzeug.exceptions import abort

from projtuto.auth import login_required
from projtuto.db import get_db
```

Nesta parte:

```py
# rota controle index
@bp.route('/')
def index():
    db = get_db()
    posts = db.execute(
        'SELECT p.id, title, body, created, author_id, username'
        ' FROM post p JOIN user u ON p.author_id = u.id'
        ' ORDER BY created DESC'
    ).fetchall()
    return render_template('blog/index.html', posts=posts)
```

Temos a rota / que leva a pagina index principal, o que o controlador fara é, mostrar na tela, todos os post feitos, por meio de um select. A variavel posts sera usada dentro da pagina html para chamar os dados da tabela do db.

Notemos que temos duas tabelas ligadas por um join, a tabela de login e de posts.

Pegando uma parte do html de index, vemos que:

```jinja2
{% block content %}
  {% for post in posts %}
    <article class="post">
      <header>
        <div>
          <h1>{{ post['title'] }}</h1>
          <div class="about">by {{ post['username'] }} on {{ post['created'].strftime('%Y-%m-%d') }}</div>
        </div>
        {% if g.user['id'] == post['author_id'] %}
          <a class="action" href="{{ url_for('blog.update', id=post['id']) }}">Edit</a>
        {% endif %}
      </header>
      <p class="body">{{ post['body'] }}</p>
    </article>
    {% if not loop.last %}
      <hr>
    {% endif %}
  {% endfor %}
{% endblock %}
```

Notemos o for feito no jinja para colocar codigos python em html. Neste for, criamos uma variavel post em posts no qual posts é aquela variavel do controlador ligada ao db. Em php esse é o forech, onde pegaremos todas as informacoes de uma tabela.

Notemos que o title esta dentro de um array de post, variavel que referencia posts, variavel do cotrolador ligada a tabela da db. Por isso podemos pegar infos da tabela e colocar no html.

Notemos o g.user, serve para pegar a info do id da sessao e confirmar se é o mesmo do autor do post, pois, um autor A nao pode editar o post do autor B, pois o id de A é distinto de B.

No botao de update, notemos que o linque da url tem o nome da rota.metodo, para levar ao metodo controlador correto. E tambem, levamos o id para esse controlador.

### criar

```py
# rota controle criar
@bp.route('/create', methods=('GET', 'POST'))
@login_required
def create():
    if request.method == 'POST':
        title = request.form['title']
        body = request.form['body']
        error = None

        if not title:
            error = 'Title is required.'

        if error is not None:
            flash(error)
        else:
            db = get_db()
            db.execute(
                'INSERT INTO post (title, body, author_id)'
                ' VALUES (?, ?, ?)',
                (title, body, g.user['id'])
            )
            db.commit()
            return redirect(url_for('blog.index'))

    return render_template('blog/create.html')
```

O metodo de criar vamos até a pagina para criar um post, pegamos do formulario o titulo e corpo do post, indicamos que um post nao pode ser ausente de titulo, caso nao tenho erro de ingresso, insertamos na tabela de post, commitamos ao db e vamos a tela de index.

```jinja2
{% block content %}
  <form method="post">
    <label for="title">Title</label>
    <input name="title" id="title" value="{{ request.form['title'] }}" required>
    <label for="body">Body</label>
    <textarea name="body" id="body">{{ request.form['body'] }}</textarea>
    <input type="submit" value="Save">
  </form>
{% endblock %}
```

Um pedaco do html de create, vemos um formulario, no qual colocamos as informacoes com o request form em value, para pegar o valor e levar ao controlador.

### update

```py
# rota controle atualizar identificar id autor
def get_post(id, check_author=True):
    post = get_db().execute(
        'SELECT p.id, title, body, created, author_id, username'
        ' FROM post p JOIN user u ON p.author_id = u.id'
        ' WHERE p.id = ?',
        (id,)
    ).fetchone()

    if post is None:
        abort(404, "Post id {0} doesn't exist.".format(id))

    if check_author and post['author_id'] != g.user['id']:
        abort(403)

    return post
```

Ao fazer um update num post, identificamos se o id do autor que quer editar, existe na tabela associada ao post.

```py
@bp.route('/<int:id>/update', methods=('GET', 'POST'))
@login_required
def update(id):
    post = get_post(id)

    if request.method == 'POST':
        title = request.form['title']
        body = request.form['body']
        error = None

        if not title:
            error = 'Title is required.'

        if error is not None:
            flash(error)
        else:
            db = get_db()
            db.execute(
                'UPDATE post SET title = ?, body = ?'
                ' WHERE id = ?',
                (title, body, id)
            )
            db.commit()
            return redirect(url_for('blog.index'))

    return render_template('blog/update.html', post=post)
```

A parte do update se parece ao de criar, a diferenca é que fazemos um update na tabela, comitamos e voltamos a pagina de index.

### Deletar

```py
# rota controle deletar
@bp.route('/<int:id>/delete', methods=('POST',))
@login_required
def delete(id):
    get_post(id)
    db = get_db()
    db.execute('DELETE FROM post WHERE id = ?', (id,))
    db.commit()
    return redirect(url_for('blog.index'))
```

Para deletar, pegamos o id de tal post, e executamos o delete do sql.
