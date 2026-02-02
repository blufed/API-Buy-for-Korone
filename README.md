// ==UserScript==
// @name         API Buy REVAMPED
// @namespace    ViolentMonkey Scripts
// @version      1.152
// @description  direct api call for purchasing
// @author       LMoD
// @match        https://www.pekora.zip/*
// @grant        none
// @run-at       document-end
// ==/UserScript==

(function () {
    'use strict';

    let observer = null;
    let retryTimer = null;

    let lastUrl = location.href;

    const info = {
        version: '1.152',
        author: '@pythonplugin',
    };

    // helpers
    const getItemId = () => {
        const m = window.location.pathname.match(/\/catalog\/(\d+)/);
        return m ? m[1] : null;
    };

    const getCsrf = () => {
        const cookies = document.cookie.split(';');

        for (let cookie of cookies) {
            const [name, value] = cookie.trim().split('=');

            if (name.toLowerCase().includes('csrf')) return value;
        }

        const meta = document.querySelector('meta[name="csrf-token"]');
        return meta ? meta.getAttribute('content') : null;
    };

    const getPrice = () => {
        const label = document.querySelector('.priceLabel-0-2-61') ||
                      document.querySelector('[class*="priceLabel"]');
        if (!label) return 0;

        const text = label.textContent.trim();

        const cleaned = text.replace(/[^\d]/g, '');
        return cleaned ? parseInt(cleaned, 10) : 0;
    };

    const getSellerId = () => {
        const sellerLink = document.querySelector('a[href*="/User.aspx?ID="]');
        if (sellerLink) {
            const match = sellerLink.href.match(/ID=(\d+)/);
            if (match) return parseInt(match[1]);
        }

        return 1;
    };

    // main
    const notify = (msg, ok = true) => {
        const toast = document.createElement('div');
        toast.textContent = msg;

        toast.style.cssText = `
            position: fixed;
            bottom: -60px;
            right: 20px;
            z-index: 9999;
            background: ${ok ? 'rgba(37, 99, 235, 0.95)' : 'rgba(195, 66, 66, 0.95)'};
            backdrop-filter: blur(6px);
            color: white;
            padding: 12px 18px;
            border-radius: 8px;
            border: 1px solid ${ok ? '#1e40af' : '#9b2f2f'};
            font-family: "Gotham SSm A", "Gotham SSm B", "Helvetica Neue", Helvetica, Arial, sans-serif;
            font-size: 13px;
            font-weight: 500;
            letter-spacing: 0.2px;
            box-shadow: 0 4px 10px rgba(0,0,0,0.25);
            transition: bottom 0.5s ease, opacity 0.4s ease;
            opacity: 0;
            text-transform: lowercase;
        `;

        document.body.appendChild(toast);

        setTimeout(() => {
            toast.style.bottom = '30px';
            toast.style.opacity = '1';
        }, 50);

        setTimeout(() => {
            toast.style.bottom = '-60px';
            toast.style.opacity = '0';

            setTimeout(() => toast.remove(), 600);
        }, 3500);
    };

    const purchase = async (id, price = 0) => {
        const csrf = getCsrf();

        try {
            const res = await fetch(`https://www.pekora.zip/apisite/economy/v1/purchases/products/${id}`, {
                method: 'POST',
                mode: 'same-origin',
                credentials: 'include',

                headers: {
                    'accept': 'application/json, text/plain, */*',
                    'content-type': 'application/json;charset=UTF-8',
                    'x-csrf-token': csrf,
                    'referer': location.href,
                    'origin': 'https://www.pekora.zip'
                },

                body: JSON.stringify({
                    assetId: parseInt(id),
                    expectedPrice: price,
                    expectedSellerId: getSellerId(),
                    expectedCurrency: 1,
                    userAssetId: null
                })
            });

            const text = await res.text();

            let data = {};
            try { data = JSON.parse(text); } catch {}

            if (res.ok && data.purchased) {
                notify('purchase successful');
                setTimeout(() => location.reload(), 1500);
            } else {
                notify(`failed: ${data.reason || text}`, false);
            }
        } catch (err) {
            notify(`failed: ${err.message}`, false);
        }
    };


    const purchase_button = () => {
        const id = getItemId();
        if (!id) return false;

        if (document.querySelector('#bypbtn')) return true;

        const buyButton =
            document.querySelector('button.newBuyButton-0-2-58') ||
            document.querySelector('button[class*="newBuyButton"]') ||
            document.querySelector('button[class*="buyBtn"]');

        if (!buyButton || buyButton.disabled || buyButton.textContent.toLowerCase().includes('edit avatar')) return false;

        const container = buyButton.parentElement;
        if (!container) return false;

        const btn = document.createElement('button');
        btn.id = 'bypbtn';
        btn.type = 'button';
        btn.textContent = 'api buy';

        btn.style.cssText = `
            background: rgb(37, 99, 235) !important;
            color: white !important;
            margin-top: 6px;
            border: none;
            border-radius: 6px;
            padding: 8px 16px;
            font-family: "Gotham SSm A", "Gotham SSm B", "Helvetica Neue", Helvetica, Arial, sans-serif;
            font-size: 14px;
            font-weight: 600;
            cursor: pointer;
            width: 100%;
            height: 40px;
            text-transform: uppercase;
            letter-spacing: 0.5px;
            transition: all 0.25s ease;
            display: flex;
            align-items: center;
            justify-content: center;
            box-shadow: 0 2px 4px rgba(0,0,0,0.2);
        `;

        btn.onmouseenter = () => {
            if (!btn.disabled) {
                btn.style.background = 'rgb(30, 64, 175) !important';
                btn.style.transform = 'translateY(-1px)';
                btn.style.boxShadow = '0 3px 6px rgba(0,0,0,0.25)';
            }
        };

        btn.onmouseleave = () => {
            if (!btn.disabled) {
                btn.style.background = 'rgb(37, 99, 235) !important';
                btn.style.transform = 'translateY(0)';
                btn.style.boxShadow = '0 2px 4px rgba(0,0,0,0.2)';
            }
        };

        btn.onclick = async (e) => {
            e.preventDefault();

            if (btn.disabled) return;
            btn.disabled = true;

            btn.textContent = 'processing...';

            btn.style.background = '#888 !important';
            btn.style.cursor = 'not-allowed';
            btn.style.opacity = '0.75';
            btn.style.transform = 'scale(0.97)';
            btn.style.boxShadow = 'none';

            try {
                await purchase(getItemId(), getPrice());
            } finally {
                btn.disabled = false;

                btn.textContent = 'api buy';

                btn.style.background = 'rgb(37, 99, 235) !important';
                btn.style.cursor = 'pointer';
                btn.style.opacity = '1';
                btn.style.transform = 'scale(1)';
                btn.style.boxShadow = '0 2px 4px rgba(0,0,0,0.2)';
            }
        };

        container.appendChild(btn);
        return true;
    };

    // 🔑 SPACEBAR BUY (ADDED, DOES NOT TOUCH LOGIC)
    document.addEventListener('keydown', (e) => {
        if (e.code !== 'Space') return;
        if (!/\/catalog\/\d+/.test(location.pathname)) return;
        if (['INPUT', 'TEXTAREA'].includes(e.target.tagName)) return;

        const btn = document.querySelector('#bypbtn');
        if (!btn || btn.disabled) return;

        e.preventDefault();
        btn.click();
    });

    // init
    const start_retry = () => {
        if (retryTimer) {
            clearTimeout(retryTimer);
            retryTimer = null;
        }

        let attempts = 0;
        const maxAttempts = 20;

        const retry = () => {
            const id = getItemId();

            if (!id) {
                return;
            }

            const success = purchase_button();

            if (success) {
                return;
            }

            attempts++;

            if (attempts < maxAttempts) {
                retryTimer = setTimeout(retry, 300);
            }
        };

        retry();
    };

    const monitor_button = () => {
        if (observer) {
            observer.disconnect();
        }

        observer = new MutationObserver(() => {
            const id = getItemId();

            if (id && !document.querySelector('#bypbtn')) {
                purchase_button();
            }
        });

        observer.observe(document.body, { childList: true, subtree: true });
    };

    const history_navigation = () => {
        const pushState = history.pushState;
        const replaceState = history.replaceState;

        const handler = () => {
            const currentUrl = location.href;
            if (currentUrl === lastUrl) return;

            lastUrl = currentUrl;

            setTimeout(() => {
                start_retry();
            }, 100);
        };

        history.pushState = function (...args) {
            pushState.apply(this, args);
            handler();
        };

        history.replaceState = function (...args) {
            replaceState.apply(this, args);
            handler();
        };

        window.addEventListener('popstate', handler);
    };

    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', () => {
            start_retry();
            monitor_button();
        });
    } else {
        start_retry();
        monitor_button();
    }

    history_navigation();
})();
