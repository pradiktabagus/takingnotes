---
date: 2026-05-02
tags:
  - architecture
  - fsd
  - clean-architecture
  - front-end
  - design-pattern
---

# Clean Architecture dan Feature Sliced Design

Membangun aplikasi yang kompleks membutuhkan arsitektur perangkat lunak yang baik agar aplikasi lebih mudah untuk dikembangkan, diuji, dan dipelihara. Dua pendekatan yang populer saat ini dalam pengembangan perangkat lunak adalah Clean Architecture dan Feature Sliced Design (FSD).

## Clean Architecture

Clean Architecture diperkenalkan oleh Robert C. Martin. Ini adalah arsitektur perangkat lunak yang memisahkan logika bisnis dari detail implementasi, seperti database dan framework. Tujuannya adalah untuk membuat sistem yang mudah diuji, fleksibel, dan tahan lama. Arsitektur ini berfokus pada prinsip-prinsip berikut:

1. Independen dari Framework
2. Testable
3. Independen dari UI
4. Independen dari Database
5. Independen dari agency atau server

## Feature Sliced Design (FSD)

FSD adalah pendekatan arsitektur untuk membangun aplikasi front-end yang kompleks dengan cara memecah aplikasi menjadi irisan (slice) fitur. Setiap fitur merupakan komponen independen yang mengelola state dan logika aplikasi. FSD berfokus pada:

1. Modularitas: Memecah aplikasi menjadi beberapa modul yang independen untuk meningkatkan skalabilitas dan pengembangan yang lebih mudah.
2. Reusabilitas: Membuat komponen yang dapat digunakan kembali untuk mengurangi redudansi kode.
3. Pembagian Tanggung Jawab: Membagi tanggung jawab antara komponen untuk meningkatkan maintainabilitas.

## Kesimpulan

Clean Architecture dan FSD adalah dua pendekatan yang berbeda namun saling melengkapi dalam pengembangan perangkat lunak. Clean Architecture berfokus pada pemisahan kepentingan dan abstraksi, sementara FSD berfokus pada modularitas dan reusabilitas pada level fitur. Kombinasi keduanya dapat menghasilkan aplikasi yang lebih mudah diuji, lebih fleksibel, dan lebih mudah dipelihara.
