# Poll based futures in Rust

## Max Bruckner

---

# Callbacks

```cpp
void download(
	std::string url,
	std::function<void(std::vector<std::byte>)> callback
);
```

---

# Callbacks

```cpp
void download(
	std::string url,
	std::function<void(std::vector<std::byte>)> callback
);
```

```cpp
void play_song(std::vector<std::byte> song);

download(
	"https://example.com/song.opus",
	[](auto song) {
		play_song(song);
	});
```

---

# Komplexeres Beispiel

## 1. Bild herunterladen.
## 2. Konvertieren
## 3. Wieder hochladen

---

# Callbacks (komplexes Beispiel)

```cpp
download(
	"https://example.com/image.png"
	[](auto image) {
		convert(
			image,
			[](auto converted_image) {
				upload(
					converted_image,
					"https://example.com/image.webp",
					[]() {
						// ...
					});
			});
	});
```

---

# Future (aka Promise)

## Repräsentiert zukünftiges Ergebnis
* **Async IO**
* **Arbeit auf anderem Thread**
* **Kombinatoren**
* **Timer**

---

# Beispiel

```rust
download("https://example.com/image.png")
	.and_then(|image| convert(image)) // redundant closure, I know
	.and_then(|converted_image|
		upload(converted_image, "https://example.com/image.webp"))
	.map(|()| /* ... */ )
```

---

# Async/Await

```rust
let image = download("https://example.com/image.png").await;
let converted_image = convert(image).await;
upload(converted_image, "https://example.com/image.webp").await;
/* ... */
```

---

# Typische Implementierung
Serviervorschlag

Sugar free!
```cpp
template <typename Result, typename Arguments>
struct Future {
	void start(Arguments arguments);
	void schedule(std::function<void(Result)> action);
	/* ... */
}
```

```cpp
upload_future.schedule([]() { /* ... */});
conversion_future.schedule([](auto converted_image) {
	upload_future.start(converted_image);
});
download_future.schedule([](auto image) {
	conversion_future.start(image);
});
download_future.start(url);
```

---

# Schwierigkeiten

* **Cancelation**
* **Thread-Synchronisierung** (Wo setze ich fort)
* **Heap-Allokationen schwer vermeidbar**

---

# std::future::Future

```rust
pub trait Future {
	type Output;

	fn poll(self: Pin<&mut Self>, cx: &mut Context)
		-> Poll<Self::Output>;
}
```

```rust
pub enum Poll<T> {
	Ready(T),
	Pending,
}
```

---

# Beispiel: Async IO

```rust
fn poll(&mut self, waker: &Waker) -> Poll<Vec<u8>> {
	if not_started {
		syscall(|result| {
			self.is_finished = true;
			self.result = result;
			waker.wake();
		});
		return Pending;
	}

	if self.is_finished {
		return Ready(self.result);
	} else {
		return Pending;
	}
}
```

---

# Unser Beispiel

```rust
download(download_url)
	.and_then(|image| convert(image))
	.and_then(|converted_image| upload(converted_image, upload_url))
	.map(|()| /* ... */ )
```

```rust
let image = download(download_url).await;
let converted_image = convert(image).await;
upload(converted_image, upload_url).await;
/* ... */
```

---

# Intern (so ungefähr)

```rust
enum ImageChain {
	Downloading(DownloadFuture),
	Converting(ConversionFuture),
	Uploading(UploadFuture),
	Finished,
}
```

---

# Intern (so ungefähr)

```rust
fn poll(&mut self, waker: &Waker) -> Poll<()> {
	match self {
		Downloading(download_future) => {
			match download_future.poll(waker) {
				Ready(image) => // transition to Converting,
				Pending => return Pending, }, },
		Converting(conversion_future) => {
			match conversion_future.poll(waker) {
				Ready(image) => // transition to Uploading,
				Pending => return Pending, } },
		Uploading(upload_future) => {
			Ready(()) => /* ... */ // also transition to Finish.,
			Pending => return Pending,
		},
		Finished => panic!("invalid"),
	}
}
```

---

# Task (in der Regel)

* **Arbeitseinheit aus vielen Teil-Futures**
* **Erhält callbacks über Waker**
* **Scheduling in Task-Queue**
* **Idr. als Einheit allokiert**
* **Pollt Top-Level-Future nach wakeup**

## Ist selbst eine Future

---

# In unserem Beispiel:

## Ein Task pro Bild-URL
## Futures für z.B.
* **TCP-Verbindung**
* **TLS-Handshake**
* **Umrechnung**
* **usw.**

---

# Fazit
## Cancelation
* **Einfach nicht mehr pollen**
## Thread-Synchronisierung
* **Nur wo wirklich nötig (e.g. nicht für Combinators)**
## Heap-Allokationen
* **Meistens nur einmal pro Task**
## Und
* **Compiler kann viel weg optimieren**

---

# Quellen


* [RustFest Zürich 2017 - Tokio: How we hit 88mph by Alex Crichton](https://www.youtube.com/watch?v=4QZ0-vIIFug)
* [std::future::Future in der Rust Standardlibrary-Dokumentation](https://doc.rust-lang.org/std/future/trait.Future.html)
* [Aaron Turon: Zero-cost futures in Rust](https://aturon.github.io/blog/2016/08/11/futures/)

# Tipp
* extundelete kann gelöschte Präsentationen wiederherstellen ...
