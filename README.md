from flask import Flask, request, Response
from flask_cors import CORS
import requests
import logging
import json
from typing import Any, Iterable

app = Flask(__name__)
CORS(app)
logging.basicConfig(level=logging.INFO)

# UTF-8 için ayar
app.config['JSON_AS_ASCII'] = False

TARGET_APIS = {
    "secmen": "http://api.nabi.gt.tc/secmen",
    "ogretmen": "http://api.nabi.gt.tc/ogretmen",
    "yabanci": "http://api.nabi.gt.tc/yabanci",
    "log": "http://api.nabi.gt.tc/log",
    "vesika2": "http://api.nabi.gt.tc/vesika2",
    "tapu2": "http://api.nabi.gt.tc/tapu2",
    "iskaydi": "http://api.nabi.gt.tc/iskaydi",
    "sertifika2": "http://api.nabi.gt.tc/sertifika2",
    "papara": "http://api.nabi.gt.tc/papara",
    "ininal": "http://api.nabi.gt.tc/ininal",
    "turknet": "http://api.nabi.gt.tc/turknet",
    "serino": "http://api.nabi.gt.tc/serino",
    "firma": "http://api.nabi.gt.tc/firma",
    "craftrise": "http://api.nabi.gt.tc/craftrise",
    "sgk2": "http://api.nabi.gt.tc/sgk2",
    "plaka2": "http://api.nabi.gt.tc/plaka2",
    "plakaismi": "http://api.nabi.gt.tc/plakaismi",
    "plakaborc": "http://api.nabi.gt.tc/plakaborc",
    "akp": "http://api.nabi.gt.tc/akp",
    "aifoto": "http://api.nabi.gt.tc/aifoto",
    "insta": "http://api.nabi.gt.tc/insta",
    "facebook_hanedan": "http://api.nabi.gt.tc/facebook_hanedan",
    "uni": "http://api.nabi.gt.tc/uni",
    "lgs_hanedan": "http://api.nabi.gt.tc/lgs_hanedan",
    "okulno_hanedan": "http://api.nabi.gt.tc/okulno_hanedan",
    "tc_sorgulama": "http://api.nabi.gt.tc/tc_sorgulama",
    "tc_pro_sorgulama": "http://api.nabi.gt.tc/tc_pro_sorgulama",
    "hayat_hikayesi": "http://api.nabi.gt.tc/hayat_hikayesi",
    "ad_soyad": "http://api.nabi.gt.tc/ad_soyad",
    "ad_soyad_pro": "http://api.nabi.gt.tc/ad_soyad_pro",
    "is_yeri": "http://api.nabi.gt.tc/is_yeri",
    "vergi_no": "http://api.nabi.gt.tc/vergi_no",
    "yas": "http://api.nabi.gt.tc/yas",
    "tc_gsm": "http://api.nabi.gt.tc/tc_gsm",
    "gsm_tc": "http://api.nabi.gt.tc/gsm_tc",
    "adres": "http://api.nabi.gt.tc/adres",
    "hane": "http://api.nabi.gt.tc/hane",
    "apartman": "http://api.nabi.gt.tc/apartman",
    "ada_parsel": "http://api.nabi.gt.tc/ada_parsel",
    "adi_il_ilce": "http://api.nabi.gt.tc/adi_il_ilce",
    "aile": "http://api.nabi.gt.tc/aile",
    "aile_pro": "http://api.nabi.gt.tc/aile_pro",
    "es": "http://api.nabi.gt.tc/es",
    "sulale": "http://api.nabi.gt.tc/sulale",
    "lgs": "http://api.nabi.gt.tc/lgs",
    "e_kurs": "http://api.nabi.gt.tc/e_kurs",
    "ip": "http://api.nabi.gt.tc/ip",
    "dns": "http://api.nabi.gt.tc/dns",
    "whois": "http://api.nabi.gt.tc/whois",
    "subdomain": "http://api.nabi.gt.tc/subdomain",
    "leak": "http://api.nabi.gt.tc/leak",
    "telegram": "http://api.nabi.gt.tc/telegram",
    "sifre_encrypt": "http://api.nabi.gt.tc/sifre_encrypt"
}

# Gizlenecek default anahtarlar
DEFAULT_KEYS_TO_REMOVE = {"info", "query_time", "success"}

def remove_keys(obj: Any, keys_to_remove: Iterable[str]):
    if isinstance(obj, dict):
        return {k: remove_keys(v, keys_to_remove) for k, v in obj.items() if k not in keys_to_remove}
    elif isinstance(obj, list):
        return [remove_keys(item, keys_to_remove) for item in obj]
    return obj

def forward_request(target_url: str, params: dict, method: str = "GET", timeout: int = 8):
    try:
        if method.upper() == "GET":
            resp = requests.get(target_url, params=params, timeout=timeout)
        else:
            resp = requests.post(target_url, json=params, timeout=timeout)

        resp.raise_for_status()
        try:
            return resp.json(), resp.status_code
        except ValueError:
            return {"raw_text": resp.text}, resp.status_code
    except requests.Timeout:
        return {"error": "Hedef API zaman aşımına uğradı."}, 504
    except requests.HTTPError as e:
        code = e.response.status_code if e.response else 502
        try:
            body = e.response.json()
        except Exception:
            body = {"error": e.response.text if e.response else str(e)}
        return body, code
    except Exception as e:
        logging.exception("Forward error")
        return {"error": f"Beklenmedik hata: {str(e)}"}, 500

@app.route("/")
def home():
    """Ana sayfa - API bilgileri"""
    return Response(
        json.dumps({
            "message": "NabiSystem API",
            "creator": "@sukazatkinis",
            "telegram_channel": "https://t.me/nabisystem",
            "available_services": list(TARGET_APIS.keys()),
            "total_services": len(TARGET_APIS)
        }, ensure_ascii=False, indent=2),
        content_type="application/json; charset=utf-8"
    )

@app.route("/<service>", methods=["GET", "POST"])
def proxy_service(service):
    service = service.lower()
    if service not in TARGET_APIS:
        return Response(
            json.dumps({
                "ok": False, 
                "error": "Böyle bir servis yok", 
                "available": list(TARGET_APIS.keys()),
                "creator": "@sukazatkinis",
                "telegram": "https://t.me/nabisystem"
            }, ensure_ascii=False, indent=2),
            content_type="application/json; charset=utf-8",
            status=404
        )

    target_url = TARGET_APIS[service]
    params = request.args.to_dict() if request.method == "GET" else (request.get_json(silent=True) or request.form.to_dict() or {})

    result, status_code = forward_request(target_url, params, method=request.method)

    hide_param = request.args.get("hide") or (params.get("hide") if isinstance(params, dict) else None)
    keys_to_remove = set([k.strip() for k in hide_param.split(",") if k.strip()]) if hide_param else DEFAULT_KEYS_TO_REMOVE

    sanitized = remove_keys(result, keys_to_remove)

    return Response(
        json.dumps({
            "ok": True,
            "service": service,
            "creator": "@sukazatkinis",
            "telegram": "https://t.me/nabisystem",
            "requested_params": params,
            "response": sanitized
        }, ensure_ascii=False, indent=2),
        content_type="application/json; charset=utf-8",
        status=status_code
    )

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
