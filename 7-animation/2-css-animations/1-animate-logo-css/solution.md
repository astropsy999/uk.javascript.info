
CSS для анімації `width` та `height`:
```css
/* original class */

#flyjet {
  transition: all 3s;
}

/* JS adds .growing */
#flyjet.growing {
  width: 400px;
  height: 240px;
}
```

Зауважте, що `transitionend` запускається двічі - один раз для кожної властивості. Тому, якщо ми не виконаємо додаткову перевірку, повідомлення з’явиться двічи
