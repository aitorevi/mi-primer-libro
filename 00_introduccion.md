Result Pattern: Cuando tus errores dejan de ser una sorpresa 🎁

El problema: Try-Catch, el amigo invisible que nadie pidió 👻



¿Alguna vez has mirado el tipo de una función en TypeScript y pensado "genial, esto retorna un User", para luego descubrir (en producción, claro) que también puede explotar en tu cara? Bienvenido al club.

typescript

async function getUserById(id: string): Promise {

const user = await database.findUser(id)

return user

}



// Más tarde, en producción...

const user = await getUserById("123") // 💥 BOOM! User not found



El tipo dice Promise. Miente. Es un mentiroso. En realidad es Promise.

El problema no es solo que pueda fallar (todas las cosas pueden fallar). El problema es que el tipo no te lo dice. TypeScript, ese amigo que tanto te ayuda a evitar bugs, aquí te deja tirado. No hay ninguna pista en el tipo que te diga "oye, quizás deberías manejar el caso de error".

Y lo peor: cuando tienes múltiples operaciones que pueden fallar, acabas con el clásico try-catch hell:

typescript

async function getUserWithPosts(userId: string) {

try {

const user = await getUser(userId)

try {

const posts = await getPosts(user.id)

try {

const comments = await getComments(posts.map(p => p.id))

return { user, posts, comments }

} catch (commentsError) {

throw new Error("Comments failed")

}

} catch (postsError) {

throw new Error("Posts failed")

}

} catch (userError) {

throw new Error("User failed")

}

}



¿Qué error fue? ¿User? ¿Posts? ¿Comments? Quién sabe. Todo es un Error genérico. Y si quieres saber qué pasó, toca parsear strings de mensajes de error como si fuera 1999.



La solución: Result Pattern, el error que no se esconde 🔦



El Result Pattern viene de lenguajes funcionales como Rust y Haskell, donde las excepciones no existen (o casi). La idea es simple y brutal:





Si tu función puede fallar, que el tipo lo diga. Explícitamente. Sin esconderse.



En lugar de retornar T y cruzar los dedos, retornas Result:

typescript

type Result =

| { ok: true; value: T }

| { ok: false; error: E }



Eso es todo. Es una unión discriminada (el ok es el discriminante). O tienes éxito con un value, o tienes un fallo con un error. Sin trucos, sin magia, sin sorpresas.



Paso 1: Define tu Result type



Primero, creamos el tipo y sus constructores:

typescript

// result.ts

export type Result =

| { ok: true; value: T }

| { ok: false; error: E }

export function success(value: T): Result {

return { ok: true, value }

}

export function failure(error: E): Result {

return { ok: false, error }

}



Ya está. Eso es el 80% del patrón. Lo demás son helpers útiles.



Paso 2: Úsalo en tus funciones



Ahora, en lugar de lanzar excepciones, retornas Result:

typescript

async function getUserById(id: string): Promise> {

try {

const user = await database.findUser(id)

return success(user) // ✅

} catch (error) {

return failureUser not found: ${id}) // ❌

}

}



¿Ves la diferencia? El tipo ya no miente. Dice claramente: "esto puede retornar un User o un string de error". TypeScript está de tu lado otra vez.



Paso 3: Maneja ambos casos



Ahora, cuando llamas a la función, TypeScript te obliga a manejar ambos casos:

typescript

const result = await getUserById("123")

if (result.ok) {

console.logWelcome, ${result.value.name}!) // TypeScript sabe que hay .value

} else {

console.errorError: ${result.error}) // TypeScript sabe que hay .error

}



No hay try-catch. No hay sorpresas. Si te olvidas del else, TypeScript te grita (bueno, te subraya en rojo, que es su forma de gritar).



Paso 4: Compón resultados sin morir en el intento



Ahora viene la magia. ¿Recuerdas el try-catch hell del inicio? Con Result se vuelve legible:

typescript

async function getUserWithPosts(

userId: string

): Promise> {

const userResult = await getUser(userId)

if (!userResult.ok) {

return failureUser error: ${userResult.error})

}

const postsResult = await getPosts(userResult.value.id)

if (!postsResult.ok) {

return failurePosts error: ${postsResult.error})

}

const commentsResult = await getComments(

postsResult.value.map(p => p.id)

)

if (!commentsResult.ok) {

return failureComments error: ${commentsResult.error})

}

return success({

user: userResult.value,

posts: postsResult.value,

comments: commentsResult.value

})

}



Cada error es específico. Cada paso es claro. No hay anidación. Es el early return pattern en todo su esplendor.

Y cuando lo usas:

typescript

const result = await getUserWithPosts("123")

if (result.ok) {

res.json(result.value)

} else {

console.error(result.error)

res.status(404).json({ error: result.error })

}



Simple y directo. El error ya viene con contexto desde la función que falló.



Bonus: Helpers que te salvan la vida 🦸



Puedes agregar funciones útiles para trabajar con Results:

typescript

// Transformar el valor solo si es success

function map(

result: Result,

fn: (value: T) => U

): Result {

if (result.ok) {

return success(fn(result.value))

}

return result

}

// Valor por defecto si falla

function getOrElse(result: Result, defaultValue: T): T {

return result.ok ? result.value : defaultValue

}

// Combinar múltiples Results

function combine(results: Result[]): Result {

const values: T[] = []

for (const result of results) {

if (!result.ok) {

return result // Primer error

}

values.push(result.value)

}

return success(values)

}



Ahora puedes hacer cosas mas locas como:

typescript

// Transformar

const userResult = await getUser("123")

const nameResult = map(userResult, user => user.name.toUpperCase())

// Valores por defecto

const user = getOrElse(

await getUser("unknown"),

{ id: "0", name: "Guest" }

)

// Combinar múltiples operaciones

const results = await Promise.all([

getUser("1"),

getUser("2"),

getUser("3")

])

const combined = combine(results)

if (combined.ok) {

console.logFound ${combined.value.length} users)

} else {

console.errorFailed: ${combined.error})

}





Testing: Donde Result brilla de verdad ✨



La verdadera ventaja en testing no es la sintaxis, sino la simetría. Con Result, success y failure son exactamente lo mismo: objetos que retornas. No hay casos especiales.

typescript

// Path feliz: seguro

it("returns_success_when_user_exists", async () => {

const result = await getUser("123")

expect(result.ok).toBe(true)

if (result.ok) {

expect(result.value.name).toBe("Alice")

}

})

// Path de error: igual de seguro

it("returns_failure_when_user_not_found", async () => {

const result = await getUser("unknown")

expect(result.ok).toBe(false)

if (!result.ok) {

expect(result.error).toContain("not found")

}

})



No hay sintaxis especial. No hay .rejects. No hay try/catch en los tests. Ambos casos son first-class citizens: solo verificas un objeto y TypeScript hace el narrowing por ti.



Conclusión final: El poder de lo explícito 💪



El Result Pattern no es magia. Es solo hacer explícito lo que siempre estuvo ahí: las funciones pueden fallar.

La diferencia es que ahora el tipo no miente, Result dice claramente que puede fallar, TypeScript te ayuda, te obliga a manejar ambos casos, el código es más claro, early returns en lugar de try-catch anidados, los tests son más simples (success y failure son iguales de importantes), y componer es fácil (helpers como map, combine, getOrElse hacen el trabajo pesado).



¿Cuándo usarlo?



Cuando manejas errores esperados de negocio (user not found, validation errors, permission denied), cuando diseñas APIs públicas donde quieres forzar a los consumidores a manejar errores.



¿Cuándo NO usarlo?



Para errores inesperados (out of memory, null pointer - esos sí que son bugs), bugs de programación (assertions), o errores críticos del sistema (database crashed, config corrupted). Esos siguen siendo excepciones legítimas.

TL;DR: Si tu función puede fallar, que el tipo lo diga. Result Pattern hace los errores explícitos, type-safe y fáciles de manejar. Tu yo del futuro (y tu equipo) te lo agradecerán. 🙏

Spoiler: ¿Conoces la monada Either? Es Result con esteroides. Quizás

tema para otro artículo... 👀



