# Demo en vivo: análisis de seguridad de un repo real con Claude Code

## Idea

Usamos el comando nativo `/security-review` de Claude Code (no requiere instalar ninguna skill externa) sobre **PyYAML**, la librería YAML más usada en Python. En vez de analizar el repo tal cual, **reintroducimos deliberadamente y en local** el patrón exacto de una vulnerabilidad real ya parcheada — **CVE-2017-18342 / CVE-2020-14343** — como si fuera un cambio a punto de commitear, y dejamos que Claude lo detecte antes de que llegue a git.

Por qué esto y no correr el análisis sobre el repo "tal cual": `/security-review` está diseñado para revisar **cambios pendientes** (el diff de lo que aún no se ha commiteado), igual que haría en un PR real. Sobre un repo recién clonado sin cambios no hay nada que revisar. Simulando el bug como un cambio sin commitear reproducimos exactamente el caso de uso real: "el desarrollador está a punto de introducir esto, ¿lo pilla la IA antes del merge?"

El bug: entre 2014 y 2019, `yaml.load(datos)` en PyYAML ejecutaba código arbitrario si el YAML venía de una fuente no confiable, porque el "Loader" por defecto podía construir cualquier objeto Python (incluida ejecución de código) en lugar de limitarse a tipos básicos. Se le asignó CVE-2017-18342 (y una variante, CVE-2020-14343). Está documentado y confirmado en Snyk, CVE Details y varios análisis técnicos.

Hoy PyYAML ya no permite ese fallo por defecto (a partir de la versión 5.1 empezó a exigir el parámetro `Loader` explícitamente). Nosotros deshacemos ese arreglo en una copia local, sin commitear, solo para la demo.

## Preparación 

```bash
git clone --depth 1 https://github.com/yaml/pyyaml.git
cd pyyaml
```

Confirma que tienes Claude Code instalado y actualizado (`claude update`) y ábrelo dentro de la carpeta `pyyaml`.

Localiza la función `load` en `lib/yaml/__init__.py`. En la versión actual del repo exige el Loader de forma explícita (ya está arreglado):

```python
def load(stream, Loader):
    """
    Parse the first YAML document in a stream
    and produce the corresponding Python object.
    """
    loader = Loader(stream)
    try:
        return loader.get_single_data()
    finally:
        loader.dispose()
```

## Guion en vivo

**1. Muestra el estado limpio del repo** (opcional, para contexto): `git status` → sin cambios.

**2. Introduce en directo, delante del público, el cambio que reintroduce el CVE.** Edita `lib/yaml/__init__.py` y cambia solo la firma de `load`:

```python
def load(stream, Loader=Loader):
```

(Es decir: le das un valor por defecto al parámetro `Loader`, usando la clase `Loader` — la variante insegura, capaz de construir cualquier objeto Python — en vez de `SafeLoader`.) Esto es literalmente el patrón que causó CVE-2017-18342: cualquier código que llame a `yaml.load(datos_no_confiables)` sin especificar Loader vuelve a ser vulnerable a ejecución remota de código.

**3. Enseña el diff:** `git diff` — una sola línea cambiada, muy poco vistosa, fácil de que pase desapercibida en una revisión humana rápida de PR. Ese es justo el punto de la demo.

**4. Lanza el análisis:** dentro de Claude Code, ejecuta:

```
/security-review
```

**5. Resultado.** Claude analiza el cambio pendiente y debería identificar como hallazgo de severidad alta algo del tipo: *"deserialización insegura / ejecución de código arbitrario: `load()` ahora usa por defecto un Loader que permite construir objetos Python arbitrarios (tag `!!python/object`) a partir de YAML no confiable"*, con explicación del impacto y sugerencia de usar `SafeLoader` o mantener `Loader` como parámetro obligatorio.

**6. Cierre:** este exacto patrón fue un CVE real, público, que tardó tiempo en descubrirse y parchearse en el ecosistema Python. Aquí la IA lo señala en segundos, sobre una única línea de diff, antes de que llegue a un commit.

## Vuelta al estado limpio

```bash
git checkout -- lib/yaml/__init__.py
```



## Fuentes

- [CVE-2017-18342 — CVE Details](https://www.cvedetails.com/cve/CVE-2017-18342/)
- [Deserialization of Untrusted Data in pyyaml — Snyk](https://security.snyk.io/vuln/SNYK-UBUNTU1404-PYYAML-287531)
- [Automated Security Reviews in Claude Code — Claude Help Center](https://support.claude.com/en/articles/11932705-automated-security-reviews-in-claude-code)
- [anthropics/claude-code-security-review — README](https://github.com/anthropics/claude-code-security-review)
- Código fuente verificado directamente en `github.com/yaml/pyyaml` (`lib/yaml/__init__.py`, `lib/yaml/loader.py`)
