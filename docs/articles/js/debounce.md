# 函数消抖与函数节流

## 函数消抖
连续的函数调用，只在最后一次调用过后的指定时间过后执行该函数

    function debounce(fn, delay) {
        let timer = null;
        return (...args) => {
            clearTimeout(timer);
            timer = setTimeout(() => {
                fn.apply(this, args);
            }, delay);
        }
    }

## 函数节流
连续的函数调用，在指定的时间内只会执行一次

    function throttle(fn, delay) {
        let timer = null;
        return (...args) => {
            if(!timer) {
                timer = setTimeout(() => { 
                    timer = null; 
                    fn.apply(this, args);
                }, delay);
            }
        }
    }
完。