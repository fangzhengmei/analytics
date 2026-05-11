# Plausible Analytics 缁村害瑙ｆ瀽涓庡睍绀哄垎鏋愭姤鍛?
## 鐩綍
1. [姒傝堪](#1-姒傝堪)
2. [Geo 瀹氫綅缁村害](#2-geo-瀹氫綅缁村害)
3. [璁惧绫诲瀷缁村害](#3-璁惧绫诲瀷缁村害)
4. [娴侀噺鏉ユ簮缁村害](#4-娴侀噺鏉ユ簮缁村害)
5. [鏁版嵁瀛樺偍妯″瀷](#5-鏁版嵁瀛樺偍妯″瀷)
6. [鏌ヨ涓庡睍绀烘祦绋媇(#6-鏌ヨ涓庡睍绀烘祦绋?
7. [鍏抽敭鏂囦欢绱㈠紩](#7-鍏抽敭鏂囦欢绱㈠紩)

---

## 1. 姒傝堪

Plausible Analytics 閲囩敤瀹屾暣鐨勬暟鎹閬撴潵澶勭悊涓変釜鏍稿績缁村害鏁版嵁锛?
**瀹屾暣澶勭悊娴佺▼锛?*
```
Tracker Script 鈫?HTTP Request 鈫?Ingestion Pipeline 鈫?ClickHouse Storage 
    鈫?SQL Query Builder 鈫?API Response 鈫?React Dashboard
```

涓変釜缁村害鐨勬暟鎹鐞嗗垎涓轰袱涓樁娈碉細
- **Geo 瀹氫綅** 鍜?**璁惧绫诲瀷** 鍦?**鏁版嵁鎽勫叆闃舵锛圛ngestion锛?* 瀹屾垚瑙ｆ瀽
- **娴侀噺鏉ユ簮锛圫ource锛?* 鍦ㄦ憚鍏ラ樁娈佃В鏋愶紝浣?**娴侀噺娓犻亾锛圕hannel锛?* 鍦?**ClickHouse 鍐欏叆鏃?* 閫氳繃 MATERIALIZED 鍒楄嚜鍔ㄨ绠?
瑙ｆ瀽鍚庝互瑙勮寖鍖栫殑瀛楁瀛樺偍鍒?ClickHouse 涓紝鏌ヨ鏃剁洿鎺ヤ娇鐢ㄥ凡璁＄畻鐨勫瓧娈佃繘琛岃仛鍚堛€?
---

## 2. Geo 瀹氫綅缁村害

### 2.1 鍦扮悊浣嶇疆瑙ｆ瀽鏈哄埗

**鏍稿績妯″潡锛?* `lib/plausible/ingestion/geolocation.ex`

#### 鍦扮悊浣嶇疆鏁版嵁婧?浣跨敤 `locus` 搴撳姞杞?MaxMind 鎴?DB-IP 鐨?MMDB 鏁版嵁搴撴枃浠讹細
- `lib/plausible/geo.ex:44-74` 鎻愪緵鏁版嵁搴撳姞杞?API
- 鏀寔涓ょ妯″紡锛?  1. **鏈湴鏂囦欢妯″紡**锛氶€氳繃 `:path` 鍙傛暟鎸囧畾 `.mmdb` 鏂囦欢
  2. **MaxMind 鎺堟潈妯″紡**锛氶€氳繃 `:license_key` + `:edition` 浠?MaxMind 涓嬭浇

#### 鍦扮悊鏁版嵁瑙ｆ瀽娴佺▼

```
IP鍦板潃 鈫?Plausible.Geo.lookup(ip) 鈫?MMDB鏌ヨ 鈫?缁撴瀯鍖栧湴鐞嗕俊鎭?```

**鍏抽敭澶勭悊姝ラ锛?* `lib/plausible/ingestion/geolocation.ex:4-52`

```elixir
def lookup(ip_address) do
  case Plausible.Geo.lookup(ip_address) do
    %{} = entry ->
      %{
        country_code: entry |> get_in(["country", "iso_code"]) |> ignore_unknown_country(),
        subdivision1_code: subdivision1_code(country_code, entry),  # 涓€绾ц鏀垮尯鍒?        subdivision2_code: subdivision2_code(country_code, entry),  # 浜岀骇琛屾斂鍖哄垝
        city_geoname_id: city_geoname_id  # 鍩庡競 Geoname ID
      }
    nil -> nil
  end
end
```

#### 鍥藉浠ｇ爜杩囨护瑙勫垯
鐗规畩鍥藉/鍦板尯浠ｇ爜浼氳杩囨护鎺夛細
- `ZZ` - Worldwide锛堝叏鐞冿級
- `XX` - Disputed territory锛堜簤璁湴鍖猴級
- `T1` - Tor exit node锛圱or 鍑哄彛鑺傜偣锛?
#### 鍖垮悕 VPN IP 澶勭悊
`lib/plausible/ingestion/event.ex:324-333`

```elixir
defp put_geolocation(%__MODULE__{} = event, _context) do
  case event.request.ip_classification do
    "anonymous_vpn_ip" ->
      update_session_attrs(event, %{country_code: "A1"})  # 鍖垮悕浠ｇ悊鏍囪涓?A1
    _any ->
      result = Plausible.Ingestion.Geolocation.lookup(event.request.remote_ip) || %{}
      update_session_attrs(event, result)
  end
end
```

**IP 鍒嗙被鏉ユ簮**锛氫粠 HTTP Header `x-plausible-ip-type` 鑾峰彇锛岀敱杈圭紭灞傦紙濡?Cloudflare锛夋敞鍏ャ€?
### 2.2 鍦扮悊鍚嶇О鏄犲皠

鍦扮悊浠ｇ爜锛坄country_code`, `subdivision_code`, `city_geoname_id`锛夊湪鏌ヨ鏃堕€氳繃 **ClickHouse Dictionary** 鏄犲皠涓哄彲璇诲悕绉般€?
**浣嶇疆鏁版嵁琛細** `lib/plausible/clickhouse_location_data.ex`

```elixir
schema "location_data" do
  field :type, Ch, type: "LowCardinality(String)"  # country / region / city
  field :id, :string                               # 浠ｇ爜/ID
  field :name, :string                             # 鏄剧ず鍚嶇О
end
```

璇ヨ〃閫氳繃 `location_data_dictionary` 瀛楀吀鍦?ALIAS 鍒椾腑琚棿鎺ヨ闂細
- `sessions_v2.country` 鈫?ALIAS 鏄犲皠鍒板浗瀹跺悕绉?- `sessions_v2.region` 鈫?ALIAS 鏄犲皠鍒板湴鍖哄悕绉?- `sessions_v2.city` 鈫?ALIAS 鏄犲皠鍒板煄甯傚悕绉?
### 2.3 Geo 鏁版嵁灞曠ず

#### 鍓嶇缁勪欢
`assets/js/dashboard/stats/locations/index.js`

**灞曠ず妯″紡锛?*
1. **鍦板浘瑙嗗浘锛圡ap锛?*锛氫娇鐢?`CountriesMap` 缁勪欢灞曠ず涓栫晫鍦板浘
2. **鍥藉鍒楄〃锛圕ountries锛?*锛氬垪琛ㄥ睍绀猴紝闄勫甫鍥芥棗
3. **鍦板尯鍒楄〃锛圧egions锛?*锛氬窞/鐪佺骇鍒?4. **鍩庡競鍒楄〃锛圕ities锛?*锛氬煄甯傜骇鍒?
#### 鍚庣 API
`lib/plausible_web/controllers/api/stats_controller.ex:751-912`

**鎺ュ彛璇存槑锛?*

| 鎺ュ彛 | 缁村害 | 鏁版嵁杞崲 |
|------|------|----------|
| `GET /countries` | `visit:country` | country_code 鈫?name + flag + alpha_3 |
| `GET /regions` | `visit:region` | subdivision_code 鈫?name + country_flag |
| `GET /cities` | `visit:city` | geoname_id 鈫?name + country_flag |

**鍥藉淇℃伅鑾峰彇锛?* 浣跨敤 `Location` 搴擄紙鍩轰簬 `:location` hex 鍖咃級鑾峰彇鍥藉鐨勮缁嗕俊鎭細
- 鍥藉鍚嶇О锛堝璇█鏀寔锛?- 鍥芥棗 emoji
- Alpha-2 / Alpha-3 浠ｇ爜

### 2.4 Geo 鏁版嵁灞忚斀瑙勫垯
`lib/plausible/ingestion/event.ex:335-360`

鏀寔鍩轰簬鍥藉浠ｇ爜鐨勫睆钄斤紙Shield 鍔熻兘锛夛細
```elixir
defp drop_shield_rule_country(event, _context) do
  if Plausible.Shields.country_blocked?(domain, country_code) do
    drop(event, :site_country_blocklist)
  else
    event
  end
end
```

---

## 3. 璁惧绫诲瀷缁村害

### 3.1 User Agent 瑙ｆ瀽鏈哄埗

**鏍稿績妯″潡锛?* `lib/plausible/ingestion/event.ex:256-551`

浣跨敤 `UAInspector` 搴撹繘琛?User Agent 瑙ｆ瀽锛岄厤鍚堝紓姝ヤ换鍔″拰瓒呮椂淇濇姢銆?
#### UA 瑙ｆ瀽娴佺▼
```
User-Agent Header 鈫?UAInspector.parse() 鈫?{os, browser, device}
```

**鍏抽敭瀹炵幇锛?* `lib/plausible/ingestion/event.ex:444-470`

```elixir
@parse_user_agent_timeout 200  # 200ms 瓒呮椂淇濇姢

defp parse_user_agent_safe(user_agent) do
  task = Task.Supervisor.async_nolink(
    Plausible.UserAgentParseTaskSupervisor,
    fn -> UAInspector.parse(user_agent) end
  )

  case Task.yield(task, @parse_user_agent_timeout) || Task.shutdown(task) do
    {:ok, result} -> {:ok, result}
    nil -> 
      emit_telemetry_ua_parse_timeout()
      {:error, :timeout}
  end
end
```

**缂撳瓨鏈哄埗锛?* 瑙ｆ瀽缁撴灉浼氱紦瀛樺湪 `:user_agents` Cache 涓紝閬垮厤閲嶅瑙ｆ瀽銆?
#### Bot 妫€娴嬩笌杩囨护
`lib/plausible/ingestion/event.ex:256-276`

浠ヤ笅 User Agent 浼氳鏍囪涓?bot 骞朵涪寮冿細
1. `UAInspector.Result.Bot{}` - 鏄庣‘鐨勬満鍣ㄤ汉
2. `Headless Chrome` - 鏃犲ご娴忚鍣?
#### 璁惧瀛楁瑙ｆ瀽

**鎿嶄綔绯荤粺锛圤perating System锛夛細** `lib/plausible/ingestion/event.ex:526-531`
```elixir
defp os_name(ua) do
  case ua.os do
    :unknown -> ""
    os -> os.name
  end
end
```

**娴忚鍣紙Browser锛夛細** `lib/plausible/ingestion/event.ex:472-488`

绉诲姩绔祻瑙堝櫒浼氳瑙勮寖鍖栵細
| 鍘熷鍚嶇О | 瑙勮寖鍖栧悗 |
|---------|---------|
| Mobile Safari | Safari |
| Chrome Mobile | Chrome |
| Chrome Mobile iOS | Chrome |
| Firefox Mobile | Firefox |
| Chrome Webview / mobile app | Mobile App |

**灞忓箷灏哄/璁惧绫诲瀷锛圫creen Size锛夛細** `lib/plausible/ingestion/event.ex:490-516`

鍩轰簬 `UAInspector.Result.Device.type` 鍒嗙被锛?
```elixir
@mobile_types ["smartphone", "feature phone", "portable media player", 
               "phablet", "wearable", "camera"]
@tablet_types ["car browser", "tablet"]
@desktop_types ["tv", "console", "desktop"]

defp screen_size(ua) do
  case ua.device do
    %Device{type: t} when t in @mobile_types -> "Mobile"
    %Device{type: t} when t in @tablet_types -> "Tablet"
    %Device{type: t} when t in @desktop_types -> "Desktop"
    _ -> nil
  end
end
```

**鐗堟湰鍙峰鐞嗭細** `lib/plausible/ingestion/event.ex:540-551`
```elixir
defp major_minor(version) do
  version
  |> String.split(".")
  |> Enum.take(2)      # 鍙彇涓荤増鏈?娆＄増鏈?  |> Enum.join(".")
end
```

### 3.2 璁惧鏁版嵁灞曠ず

#### 鍓嶇缁勪欢
`assets/js/dashboard/stats/devices/index.js`

**涓変釜 Tab 瑙嗗浘锛?*

1. **娴忚鍣紙Browsers锛?*
   - 缁村害锛歚visit:browser`
   - 鏄剧ず娴忚鍣ㄥ浘鏍囷紙Chrome, Safari, Firefox, Edge 绛夛級
   - 鐐瑰嚮鍙笅閽诲埌鐗堟湰鏄庣粏

2. **鎿嶄綔绯荤粺锛圤perating Systems锛?*
   - 缁村害锛歚visit:os`
   - 鏄剧ず OS 鍥炬爣锛圵indows, macOS, iOS, Android 绛夛級
   - 鐐瑰嚮鍙笅閽诲埌鐗堟湰鏄庣粏

3. **璁惧绫诲瀷锛圖evices锛?*
   - 缁村害锛歚visit:device`锛堝嵆 screen_size锛?   - 鏄剧ず SVG 鍥炬爣锛歁obile / Tablet / Desktop

#### 鍚庣 API
`lib/plausible_web/controllers/api/stats_controller.ex:915-1121`

| 鎺ュ彛 | 缁村害 | 璇存槑 |
|------|------|------|
| `GET /browsers` | `visit:browser` | 娴忚鍣ㄥ垪琛?|
| `GET /browser-versions` | `visit:browser_version` | 鐗堟湰鏄庣粏锛堝悓鏃惰繑鍥?browser 鍒嗙粍锛?|
| `GET /operating-systems` | `visit:os` | 鎿嶄綔绯荤粺鍒楄〃 |
| `GET /operating-system-versions` | `visit:os_version` | 绯荤粺鐗堟湰鏄庣粏 |
| `GET /screen-sizes` | `visit:device` | 璁惧绫诲瀷锛圡obile/Tablet/Desktop锛?|

**鐗堟湰鏌ヨ鐗规畩澶勭悊锛?* `lib/plausible/stats/breakdown.ex:135-139`
```elixir
def transform_dimensions("visit:browser_version"),
  do: ["visit:browser", "visit:browser_version"]

def transform_dimensions("visit:os_version"), 
  do: ["visit:os", "visit:os_version"]
```
鏌ヨ鐗堟湰鏃朵細鍚屾椂鏌ヨ涓诲悕绉帮紝鐢ㄤ簬鍓嶇鏄剧ず缁勫悎鍚嶇О銆?
---

## 4. 娴侀噺鏉ユ簮缁村害

### 4.1 鏉ユ簮瑙ｆ瀽鏈哄埗

**鏍稿績妯″潡锛?* 
- `lib/plausible/ingestion/source.ex` - Source 瑙ｆ瀽
- `lib/plausible/ingestion/acquisition.ex` - Channel 鎺ㄦ柇

#### 鏉ユ簮瑙ｆ瀽浼樺厛绾?`lib/plausible/ingestion/source.ex:66-80`

```elixir
def resolve(request) do
  tagged_source =
    request.query_params["utm_source"] ||
      request.query_params["source"] ||
      request.query_params["ref"]

  source =
    cond do
      tagged_source -> tagged_source                           # 1. UTM 鍙傛暟浼樺厛
      has_valid_referral?(request) -> parse(request.referrer)  # 2. Referer 澶?      true -> nil
    end

  find_mapping(source)  # 3. 瑙勮寖鍖栨槧灏?end
```

#### Referer 瑙ｆ瀽
`lib/plausible/ingestion/source.ex:82-100`

浣跨敤 `RefInspector` 搴撹В鏋?Referer锛?```elixir
def parse(ref) do
  case RefInspector.parse(ref).source do
    :unknown ->
      uri = URI.parse(String.trim(ref))
      format_referrer_host(uri)   # 鏈煡鏉ユ簮鐢ㄥ煙鍚?    source ->
      source                      # 宸茬煡鏉ユ簮锛堝 "Google"锛?  end
end
```

**鏉ユ簮瑙勮寖鍖栨槧灏勶細**
- 鍐呯疆 `RefInspector` 鏁版嵁搴擄紙`priv/ref_inspector/referers.yml`锛?- 鑷畾涔夋墿灞?`priv/custom_sources.json`
- 鏀寔澶у皬鍐欎笉鏁忔劅鍖归厤

**鑷畾涔夋潵婧愮ず渚嬶細**
| 缂╁啓/鍙樹綋 | 瑙勮寖鍖栧悗 |
|----------|---------|
| ig | Instagram |
| adwords | Google |
| yt-ads | YouTube |

#### 鏈夋晥 Referer 妫€鏌?`lib/plausible/ingestion/source.ex:110-124`

蹇呴』婊¤冻浠ヤ笅鏉′欢鎵嶈涓烘槸鏈夋晥寮曠敤锛?1. Scheme 涓?`http` / `https` / `android-app`
2. Host 涓嶄负绌?3. **涓嶆槸鍐呴儴鍩熷悕**锛堜笌褰撳墠绔欑偣 hostname 瀵规瘮锛?4. **涓嶆槸 localhost**

#### 娴侀噺鑾峰彇娓犻亾鎺ㄦ柇

**鏍稿績鏈哄埗锛歁ATERIALIZED 鍒?+ ClickHouse 鍑芥暟**

`acquisition_channel` 鏄?**MATERIALIZED 鍒?*锛屽湪鏁版嵁鍐欏叆鏃剁敱 ClickHouse 鑷姩璁＄畻骞舵寔涔呭寲瀛樺偍銆?
**1. ClickHouse 鍑芥暟瀹氫箟**
`priv/data_migrations/AcquisitionChannel/sql/acquisition_channel_functions.sql.eex:198-230`

```sql
CREATE OR REPLACE FUNCTION acquisition_channel AS
(referrer_source, utm_medium, utm_campaign, utm_source, click_id_param) ->
    acquisition_channel_lowered(
        lower(referrer_source),
        lower(utm_medium),
        lower(utm_campaign),
        lower(utm_source),
        click_id_param
    );

CREATE OR REPLACE FUNCTION acquisition_channel_lowered AS
(referrer_source, utm_medium, utm_campaign, utm_source, click_id_param) ->
    multiIf(
        acquisition_channel_cross_network(utm_campaign), 'Cross-network',
        acquisition_channel_display(utm_medium), 'Display',
        acquisition_channel_paid_shopping(referrer_source, utm_medium, utm_campaign), 'Paid Shopping',
        acquisition_channel_paid_search(referrer_source, utm_medium, utm_source, click_id_param), 'Paid Search',
        acquisition_channel_paid_social(referrer_source, utm_medium, utm_source), 'Paid Social',
        acquisition_channel_paid_video(referrer_source, utm_medium, utm_source), 'Paid Video',
        acquisition_channel_paid_medium(utm_medium), 'Paid Other',
        acquisition_channel_organic_shopping(referrer_source, utm_campaign), 'Organic Shopping',
        acquisition_channel_organic_social(referrer_source, utm_medium), 'Organic Social',
        acquisition_channel_organic_video(referrer_source, utm_medium), 'Organic Video',
        acquisition_channel_has_category_search(referrer_source), 'Organic Search',
        acquisition_channel_email(referrer_source, utm_source, utm_medium), 'Email',
        acquisition_channel_affiliates(utm_medium), 'Affiliates',
        acquisition_channel_audio(utm_medium), 'Audio',
        acquisition_channel_sms(utm_source), 'SMS',
        acquisition_channel_sms(utm_medium), 'SMS',
        acquisition_channel_mobile_push_notifications(utm_medium, referrer_source), 'Mobile Push Notifications',
        acquisition_channel_referral(utm_medium, referrer_source), 'Referral',
        'Direct'
    );
```

**2. 琛ㄥ垪瀹氫箟 - MATERIALIZED**
`priv/data_migrations/AcquisitionChannel/sql/acquisition_channel_add_materialized_column.sql.eex:1-4`

```sql
ALTER TABLE sessions_v2
ADD COLUMN IF NOT EXISTS acquisition_channel LowCardinality(String)
MATERIALIZED acquisition_channel(referrer_source, utm_medium, utm_campaign, utm_source, click_id_param)
```

**3. 鍐欏叆鏃舵帓闄?*
`lib/plausible/ingestion/write_buffer.ex:153`

```elixir
defp fields_to_ignore(), do: [:acquisition_channel, :interactive?]
```

鍐欏叆鏃朵笉鍖呭惈 `acquisition_channel` 瀛楁锛孋lickHouse 浼氭牴鎹鏁版嵁鑷姩璁＄畻骞跺瓨鐩樸€?
**4. 娓犻亾鍒嗙被瀹屾暣鍒楄〃锛堜紭鍏堢骇浠庨珮鍒颁綆锛屽叡 19 绫伙級**

| 搴忓彿 | 娓犻亾 | 鏉′欢 |
|------|------|------|
| 1 | Cross-network | utm_campaign 鍖呭惈 "cross-network" |
| 2 | Display | utm_medium IN ('display', 'banner', 'expandable', 'interstitial', 'cpm') |
| 3 | Paid Shopping | 璐墿鏉ユ簮 + 浠樿垂 medium 鎴?shopping campaign |
| 4 | Paid Search | 鎼滅储鏉ユ簮 + 浠樿垂 medium/utm_source 鎴?gclid/msclkid |
| 5 | Paid Social | 绀句氦鏉ユ簮 + 浠樿垂 medium/utm_source |
| 6 | Paid Video | 瑙嗛鏉ユ簮 + 浠樿垂 medium/utm_source |
| 7 | **Paid Other** | utm_medium 鍖归厤 `^(.*cp.*\|ppc\|retargeting\|paid.*)$` |
| 8 | Organic Shopping | 璐墿鏉ユ簮鎴?shopping campaign |
| 9 | Organic Social | 绀句氦鏉ユ簮鎴?utm_medium 鍖呭惈 social |
| 10 | Organic Video | 瑙嗛鏉ユ簮鎴?utm_medium 鍖呭惈 video |
| 11 | Organic Search | 鎼滅储鏉ユ簮 |
| 12 | Email | 閭欢鏉ユ簮鎴?utm 鍙傛暟鍖呭惈 email 鍏抽敭璇?|
| 13 | Affiliates | utm_medium == "affiliate" |
| 14 | Audio | utm_medium == "audio" |
| 15 | SMS | utm_source/utm_medium == "sms" |
| 16 | Mobile Push Notifications | utm_medium 鍖呭惈 push/mobile/notification 鎴?source == firebase |
| 17 | Referral | utm_medium 涓?referral/app/link 鎴栨湁 source |
| 18 | Direct | 鍏朵粬鎵€鏈夋儏鍐?|

**娉ㄦ剰锛?* 姝ゅ鍏?17 绉嶆樉寮忓垎绫?+ 1 涓粯璁?Direct锛屽叡 18 涓彲杩斿洖鍊硷紙SMS 琚袱涓潯浠舵鏌ワ級銆?
**5. 鏉ユ簮绫诲埆 Dictionary**

ClickHouse 涓垱寤轰簡涓や釜 Dictionary 鐢ㄤ簬鏉ユ簮鍒嗙被锛?
- `acquisition_channel_source_category_dict`锛氭潵婧?鈫?绫诲埆锛圫EARCH/SOCIAL/SHOPPING/VIDEO/EMAIL锛?- `acquisition_channel_paid_sources_dict`锛氫粯璐规潵婧愰泦鍚?
**鑷畾涔夋潵婧愮被鍒墿灞曪細**
`lib/plausible/ingestion/acquisition.ex:20-40`

```elixir
@custom_source_categories [
  {"hacker news", "SOURCE_CATEGORY_SOCIAL"},
  {"yahoo!", "SOURCE_CATEGORY_SEARCH"},
  {"gmail", "SOURCE_CATEGORY_EMAIL"},
  {"telegram", "SOURCE_CATEGORY_SOCIAL"},
  {"slack", "SOURCE_CATEGORY_SOCIAL"},
  {"producthunt", "SOURCE_CATEGORY_SOCIAL"},
  {"github", "SOURCE_CATEGORY_SOCIAL"},
  {"perplexity", "SOURCE_CATEGORY_SEARCH"},     # AI 鎼滅储
  {"chatgpt.com", "SOURCE_CATEGORY_SEARCH"},    # AI 鎼滅储
  {"brave", "SOURCE_CATEGORY_SEARCH"},
  {"discord", "SOURCE_CATEGORY_SOCIAL"},
  {"temu.com", "SOURCE_CATEGORY_SHOPPING"},
  # ...
]
```

**鍩虹鏁版嵁婧愶細** `priv/ga4-source-categories.csv`锛堟潵鑷?Google Analytics 4 瀹樻柟鍒嗙被锛?
**6. Click ID 鑷姩鎺ㄦ柇**
`lib/plausible/ingestion/event.ex:312-322`

```elixir
defp maybe_infer_medium(%__MODULE__{} = event, _context) do
  inferred_medium =
    case event.clickhouse_session_attrs do
      %{utm_medium: medium} when is_binary(medium) -> medium
      %{utm_medium: nil, referrer_source: "Google", click_id_param: "gclid"} -> "(gclid)"
      %{utm_medium: nil, referrer_source: "Bing", click_id_param: "msclkid"} -> "(msclkid)"
      _ -> nil
    end
  update_session_attrs(event, %{utm_medium: inferred_medium})
end
```

鏀寔鐨?Click ID 鍙傛暟锛歚gclid`, `gbraid`, `wbraid`, `msclkid`, `fbclid`, `twclid`

### 4.2 娴侀噺鏉ユ簮鏁版嵁灞曠ず

#### 鍓嶇缁勪欢
`assets/js/dashboard/stats/sources/index.js`

**鍥涗釜瑙嗗浘妯″紡锛?*

1. **Channels锛堟笭閬擄級**
   - 缁村害锛歚visit:channel`
   - 灞曠ず锛歄rganic Search, Direct, Paid Search, Paid Other, Social, Referral, Email 绛?
2. **Sources锛堟潵婧愶級**
   - 缁村害锛歚visit:source`
   - 灞曠ず鏉ユ簮缃戠珯鍥炬爣锛坒avicon锛?   - 鏀寔涓嬮捇鍔熻兘

3. **UTM 鍙傛暟**
   - 涓嬫媺鑿滃崟鍖呭惈锛歶tm_medium, utm_source, utm_campaign, utm_content, utm_term

#### 瀹屾暣涓嬮捇璺緞

```
鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?                   娴侀噺鏉ユ簮瑙嗗浘                             鈹?鈹溾攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹? Channels(娓犻亾)  鈹? Sources(鏉ユ簮)  鈹? UTM(鍙傛暟)              鈹?鈹溾攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹尖攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹尖攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?                 鈹?鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?                       鈹?鈹?                 鈹?鈹? Google     鈹?鈹傗攢鈹€ 鐐瑰嚮 Google 鏉ユ簮    鈹?鈹?                 鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹?鈹?  (鐗规畩澶勭悊)           鈹?鈹?                 鈹?       鈹?       鈹?                       鈹?鈹?                 鈹?       鈻?       鈹?                       鈹?鈹?                 鈹?鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?  Search Console API  鈹?鈹?                 鈹?鈹?鍏抽敭璇嶅垪琛? 鈹?鈹?  鈫?鎼滅储鍏抽敭璇?        鈹?鈹?                 鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?                       鈹?鈹?                 鈹?                鈹?                       鈹?鈹?                 鈹?鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?                       鈹?鈹?                 鈹?鈹? Twitter    鈹?鈹傗攢鈹€ 鐐瑰嚮鍏朵粬鏉ユ簮         鈹?鈹?                 鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹攢鈹€鈹€鈹€鈹€鈹€鈹?鈹?  (閫氱敤璺緞)           鈹?鈹?                 鈹?       鈹?       鈹?                       鈹?鈹?                 鈹?       鈻?       鈹?                       鈹?鈹?                 鈹?鈹屸攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?  /referrers/:referrer  鈹?鈹?                 鈹?鈹?鍏蜂綋referrer鈹?鈹?  鈫?寮曠敤鍩熷悕鍒楄〃       鈹?鈹?                 鈹?鈹?(twitter.com 鈹?鈹?                       鈹?鈹?                 鈹?鈹? github.com) 鈹?鈹?                       鈹?鈹?                 鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?鈹?                       鈹?鈹斺攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹粹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹粹攢鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹€鈹?```

**1. Sources 鍒楄〃缁勪欢**
`assets/js/dashboard/stats/sources/index.js`

鐐瑰嚮鏌愪釜鏉ユ簮鍚庯紝濡傛灉鏄?"Google"锛屼細鏈夌壒娈婂鐞嗭紱鍏朵粬鏉ユ簮璺宠浆鍒?ReferrerDrilldownModal銆?
**2. Referrer 涓嬮捇妯℃€佹**
`assets/js/dashboard/stats/modals/referrer-drilldown.js`

```javascript
const reportInfo = {
  title: 'Referrer drilldown',
  dimension: 'referrer',
  endpoint: url.apiPath(site, `/referrers/${url.maybeEncodeRouteParam(referrer)}`),
  dimensionLabel: 'Referrer',
  defaultOrder: ['visitors', SortDirection.desc]
}
```

**3. 鍚庣涓嬮捇 API**
`lib/plausible_web/controllers/api/stats_controller.ex:526-612`

**Google 鏉ユ簮鐗规畩澶勭悊锛?*
```elixir
def referrer_drilldown(conn, %{"referrer" => "Google"} = params) do
  # 璋冪敤 Google Search Console API 鑾峰彇鎼滅储鍏抽敭璇?  search_terms = google_api().fetch_stats(site, query, pagination, search)
  # ... 杩斿洖鍏抽敭璇嶅垪琛?end
```

**鍏朵粬鏉ユ簮閫氱敤澶勭悊锛?*
```elixir
def referrer_drilldown(conn, %{"referrer" => referrer} = params) do
  query =
    Query.from(site, params, debug_metadata: debug_metadata(conn))
    |> Query.add_filter([:is, "visit:source", [referrer]])  # 鎸夋潵婧愯繃婊?  
  # 鍒楀嚭璇ユ潵婧愪笅鐨勬墍鏈夊叿浣?referrer 鍩熷悕
  %{results: results, meta: meta} = Stats.breakdown(site, query, metrics, pagination)
end
```

#### 鍚庣 API 瀹屾暣鍒楄〃
`lib/plausible_web/controllers/api/stats_controller.ex:59-612`

| 鎺ュ彛 | 缁村害 | 璇存槑 |
|------|------|------|
| `GET /sources` | `visit:source` | 鏉ユ簮鍒楄〃锛堥《灞傦級 |
| `GET /channels` | `visit:channel` | 娓犻亾鍒楄〃锛堥《灞傦級 |
| `GET /referrers/:referrer` | `visit:referrer` | 鏉ユ簮涓嬮捇 - 鍏蜂綋寮曠敤鍩熷悕 |
| `GET /utm_mediums` | `visit:utm_medium` | UTM 濯掍粙 |
| `GET /utm_sources` | `visit:utm_source` | UTM 鏉ユ簮 |
| `GET /utm_campaigns` | `visit:utm_campaign` | UTM 骞垮憡绯诲垪 |
| `GET /utm_contents` | `visit:utm_content` | UTM 鍐呭 |
| `GET /utm_terms` | `visit:utm_term` | UTM 鍏抽敭璇?|

**涓嬮捇璺緞绀轰緥锛?*
```
/GET /sources
  鈫?鐐瑰嚮 "Twitter"
  鈹溾攢 source == "Google" 鈫?璋冪敤 Search Console API 杩斿洖鍏抽敭璇?  鈹斺攢 source != "Google" 鈫?GET /referrers/Twitter 鈫?杩斿洖 twitter.com 绛夊叿浣撳煙鍚?       鈫?鐐瑰嚮鏌愪釜鍩熷悕
       鈹斺攢 搴旂敤杩囨护鍣?"referrer is twitter.com" 鍒版暣涓华琛ㄧ洏
```

---

## 5. 鏁版嵁瀛樺偍妯″瀷

### 5.1 Session 琛ㄧ粨鏋?`lib/plausible/clickhouse_session_v2.ex`

鏍稿績缁村害瀛楁锛?
```elixir
# Geo 瀹氫綅
field :country_code, Ch, type: "LowCardinality(FixedString(2))"
field :subdivision1_code, Ch, type: "LowCardinality(String)"
field :subdivision2_code, Ch, type: "LowCardinality(String)"
field :city_geoname_id, Ch, type: "UInt32"

# 璁惧绫诲瀷
field :screen_size, Ch, type: "LowCardinality(String)"       # Mobile/Tablet/Desktop
field :operating_system, Ch, type: "LowCardinality(String)"
field :operating_system_version, Ch, type: "LowCardinality(String)"
field :browser, Ch, type: "LowCardinality(String)"
field :browser_version, Ch, type: "LowCardinality(String)"

# 娴侀噺鏉ユ簮
field :referrer_source, :string                            # 瑙勮寖鍖栨潵婧愬悕
field :referrer, :string                                    # 鍏蜂綋寮曠敤 URL
field :click_id_param, Ch, type: "LowCardinality(String)"  # gclid/msclkid 绛?field :utm_medium, :string
field :utm_source, :string
field :utm_campaign, :string
field :utm_content, :string
field :utm_term, :string
field :acquisition_channel, Ch, type: "LowCardinality(String)", writable: :never
```

**鍏抽敭鐐癸細**
- `acquisition_channel` 鏄?`writable: :never` 涓旀槸 **MATERIALIZED 鍒?*
- 鍐欏叆鏃剁敱 ClickHouse 鑷姩璁＄畻骞舵寔涔呭寲
- 鍐欏叆鏁版嵁鏃舵帓闄よ瀛楁锛坄fields_to_ignore()`锛?
### 5.2 缁村害鍚嶇О鏄犲皠

`lib/plausible/stats/sql/expression.ex:164-246`

鏌ヨ鏃剁殑缁村害閫夋嫨锛?
| 鏌ヨ缁村害 | 瀵瑰簲瀛楁 |
|---------|---------|
| `visit:country` | `t.country`锛圓LIAS 鈫?鍚嶇О锛?|
| `visit:region` | `t.region`锛圓LIAS 鈫?鍚嶇О锛?|
| `visit:city` | `t.city`锛圓LIAS 鈫?鍚嶇О锛?|
| `visit:device` | `t.device`锛堝嵆 screen_size锛?|
| `visit:os` | `t.os` |
| `visit:os_version` | `t.os_version` |
| `visit:browser` | `t.browser` |
| `visit:browser_version` | `t.browser_version` |
| `visit:source` | `t.source`锛堝嵆 referrer_source锛?|
| `visit:channel` | `t.acquisition_channel`锛圡ATERIALIZED锛屽凡瀛樺偍锛?|
| `visit:referrer` | `t.referrer` |
| `visit:utm_*` | `t.utm_*` |

绌哄€煎鐞嗭細
- Source: `"Direct / None"`
- Channel: `"Direct"`
- 鍏朵粬: `"(not set)"`

### 5.3 ClickHouse Dictionary 涓庡嚱鏁?
**1. 鍦扮悊浣嶇疆 Dictionary**
- `location_data_dictionary`锛氫唬鐮?鈫?鍚嶇О鏄犲皠

**2. 娓犻亾鍒嗙被 Dictionary锛圕lickHouse 绔級**
- `acquisition_channel_source_category_dict`锛氭潵婧?鈫?绫诲埆
- `acquisition_channel_paid_sources_dict`锛氫粯璐规潵婧愰泦鍚?
**3. 娓犻亾鎺ㄦ柇鍑芥暟锛圕lickHouse 绔級**
```sql
acquisition_channel(referrer_source, utm_medium, utm_campaign, utm_source, click_id_param)
  鈹斺攢鈫?璋冪敤 acquisition_channel_lowered锛堣浆灏忓啓鍚庡尮閰嶏級
```

---

## 6. 鏌ヨ涓庡睍绀烘祦绋?
### 6.1 瀹屾暣鏁版嵁娴?
```
鍓嶇璇锋眰
    鈫?PlausibleWeb.Api.StatsController
    鈫? (閫夋嫨 property 鍙傛暟濡?"visit:country")
Plausible.Stats.breakdown/4
    鈫?Plausible.Stats.Breakdown.breakdown/5
    鈫? (鏋勫缓 Query 缁撴瀯浣?
Plausible.Stats.QueryRunner.run/2
    鈫?Plausible.Stats.SQL.QueryBuilder.build/2
    鈫? (鐢熸垚 ClickHouse SQL)
ClickHouse 鏌ヨ
    鈫? (鎸夌淮搴?GROUP BY + 鎸囨爣鑱氬悎)
鏌ヨ缁撴灉
    鈫? (API 灞傞檮鍔犲悕绉?鍥芥棗绛夊睍绀轰俊鎭?
鍓嶇 React 缁勪欢娓叉煋
```

### 6.2 鏁版嵁鎽勫叆娴佹按绾?
`lib/plausible/ingestion/event.ex:130-151`

```elixir
defp pipeline() do
  [
    drop_verification_agent: &drop_verification_agent/2,
    drop_datacenter_ip: &drop_datacenter_ip/2,
    drop_threat_ip: &drop_threat_ip/2,
    drop_shield_rule_hostname: &drop_shield_rule_hostname/2,
    drop_shield_rule_page: &drop_shield_rule_page/2,
    drop_shield_rule_ip: &drop_shield_rule_ip/2,
    put_geolocation: &put_geolocation/2,           # 鈫?Geo 瀹氫綅
    drop_shield_rule_country: &drop_shield_rule_country/2,
    put_user_agent: &put_user_agent/2,             # 鈫?璁惧绫诲瀷
    put_basic_info: &put_basic_info/2,
    put_source_info: &put_source_info/2,           # 鈫?娴侀噺鏉ユ簮 (Source + UTM)
    maybe_infer_medium: &maybe_infer_medium/2,     # 鈫?鎺ㄦ柇 medium (Click ID)
    put_props: &put_props/2,
    put_revenue: &put_revenue/2,
    put_salts: &put_salts/2,
    put_user_id: &put_user_id/2,
    validate_clickhouse_event: &validate_clickhouse_event/2,
    register_session: &register_session/2
  ]
end
```

**娉ㄦ剰锛?* Ingestion 闃舵鍙В鏋?Source 鍜?UTM 鍙傛暟锛屼笉瑙ｆ瀽 Channel銆侰hannel 鐢?ClickHouse 鐨?MATERIALIZED 鍒楀湪鍐欏叆鏃惰嚜鍔ㄨ绠椼€?
### 6.3 娓犻亾璁＄畻鐨勫畬鏁撮摼璺?
```
Tracker 鍙戦€佽姹?    鈫?Ingestion Pipeline
    鈹溾攢 put_source_info 鈫?鍐欏叆 referrer_source, utm_*, click_id_param
    鈹斺攢 涓嶈绠?acquisition_channel
    鈫?WriteBuffer 鍐欏叆 ClickHouse
    鈹斺攢 鎺掗櫎 acquisition_channel 瀛楁 (fields_to_ignore)
    鈫?ClickHouse MATERIALIZED 鍒楄Е鍙?    鈹溾攢 璋冪敤 acquisition_channel() 鍑芥暟
    鈹溾攢 浣跨敤 Dictionary 杩涜鏉ユ簮鍒嗙被鍖归厤
    鈹斺攢 璁＄畻缁撴灉鎸佷箙鍖栧瓨鍌?    鈫?鍚庣画鏌ヨ鐩存帴璇诲彇宸插瓨鍌ㄧ殑 acquisition_channel 鍊?```

---

## 7. 鍏抽敭鏂囦欢绱㈠紩

| 鏂囦欢璺緞 | 鑱岃矗 |
|---------|------|
| `lib/plausible/ingestion/geolocation.ex` | Geo 瀹氫綅瑙ｆ瀽 |
| `lib/plausible/geo.ex` | Geo 鏁版嵁搴撳姞杞?API |
| `lib/plausible/ingestion/event.ex` | 璁惧绫诲瀷 + 鏉ユ簮瑙ｆ瀽涓诲叆鍙?|
| `lib/plausible/ingestion/source.ex` | 娴侀噺鏉ユ簮锛圫ource锛夎В鏋?|
| `lib/plausible/ingestion/acquisition.ex` | Channel 鎺ㄦ柇閫昏緫锛圗lixir 绔紝渚涜縼绉诲拰娴嬭瘯鐢級 |
| `lib/plausible/ingestion/write_buffer.ex` | 鍐欏叆缂撳啿鍖猴紝鎺掗櫎 acquisition_channel |
| `lib/plausible/ingestion/request.ex` | HTTP 璇锋眰鏋勫缓 |
| `lib/plausible/clickhouse_session_v2.ex` | Session 瀛樺偍妯″瀷 |
| `lib/plausible/data_migration/acquisition_channel.ex` | 娓犻亾瀛楁鏁版嵁杩佺Щ |
| `lib/plausible/stats/sql/expression.ex` | SQL 缁村害/鎸囨爣琛ㄨ揪寮?|
| `lib/plausible/stats/sql/query_builder.ex` | SQL 鏌ヨ鏋勫缓 |
| `lib/plausible/stats/breakdown.ex` | Breakdown 鏌ヨ灏佽 |
| `lib/plausible_web/controllers/api/stats_controller.ex` | 缁熻 API 绔偣 |
| `assets/js/dashboard/stats/locations/index.js` | 鍦扮悊浣嶇疆鍓嶇灞曠ず |
| `assets/js/dashboard/stats/devices/index.js` | 璁惧绫诲瀷鍓嶇灞曠ず |
| `assets/js/dashboard/stats/sources/index.js` | 娴侀噺鏉ユ簮鍓嶇灞曠ず |
| `assets/js/dashboard/stats/modals/referrer-drilldown.js` | 鏉ユ簮涓嬮捇妯℃€佹 |
| `priv/ref_inspector/referers.yml` | Referer 瑙ｆ瀽瑙勫垯搴?|
| `priv/custom_sources.json` | 鑷畾涔夋潵婧愭槧灏?|
| `priv/ga4-source-categories.csv` | GA4 鏉ユ簮绫诲埆鍩哄噯 |
| `priv/data_migrations/AcquisitionChannel/sql/acquisition_channel_functions.sql.eex` | ClickHouse 娓犻亾鍑芥暟瀹氫箟 |
| `priv/data_migrations/AcquisitionChannel/sql/acquisition_channel_add_materialized_column.sql.eex` | 娓犻亾 MATERIALIZED 鍒楀畾涔?|

---

## 8. 鎬荤粨

涓変釜缁村害鐨勮璁℃ā寮忛珮搴︿竴鑷翠絾鍚勬湁鐗圭偣锛?
### 8.1 鍏卞悓鐐?1. **鎽勫叆鏃惰В鏋?*锛氬湪鏁版嵁杩涘叆 ClickHouse 涔嬪墠瀹屾垚瑙ｆ瀽
2. **瑙勮寖鍖栧瓨鍌?*锛氫娇鐢ㄤ唬鐮?ID 鎴栬鑼冨寲瀛楃涓插瓨鍌?3. **鍒嗗眰灞曠ず**锛氬墠绔娇鐢?Tab 鍒囨崲涓嶅悓绮掑害鐨勮鍥?
### 8.2 鍚勭淮搴﹀樊寮?
| 缁村害 | 瀛樺偍鏂瑰紡 | 璁＄畻鏃舵満 | 鏌ヨ鏂瑰紡 |
|------|---------|---------|---------|
| **Geo 瀹氫綅** | 浠ｇ爜瀛樺偍锛坈ountry_code 绛夛級 | Ingestion 闃舵 | Dictionary + ALIAS 鍒楁槧灏勫悕绉?|
| **璁惧绫诲瀷** | 鐩存帴瀛樺偍瑙勮寖鍖栧€?| Ingestion 闃舵 | 鐩存帴鏌ヨ |
| **娴侀噺娓犻亾** | MATERIALIZED 鍒楀瓨鐩?| **ClickHouse 鍐欏叆鏃?* | 鐩存帴璇诲彇宸茶绠楀€?|

### 8.3 娓犻亾璁＄畻鐨勫叧閿璁?
`acquisition_channel` 鐨勭壒娈婅璁★細

1. **璁＄畻浣嶇疆**锛氫笉鍦?Elixir Ingestion 涓绠楋紝鑰屽湪 ClickHouse 鍐欏叆鏃堕€氳繃 MATERIALIZED 鍒楄绠?2. **璁＄畻閫昏緫**锛氶€氳繃 ClickHouse SQL 鍑芥暟 + Dictionary 瀹炵幇
3. **鍚屾鏈哄埗**锛氭暟鎹縼绉昏剼鏈細浠?Elixir 鐨?`Plausible.Ingestion.Acquisition` 妯″潡鎻愬彇瑙勫垯锛屽悓姝ュ埌 ClickHouse
4. **浼樼偣**锛?   - 鏁版嵁鍐欏叆鏃惰嚜鍔ㄨ绠楋紝Elixir 绔棤闇€鍏冲績
   - 鍙€氳繃 ClickHouse 鍑芥暟鏇存柊閫昏緫骞?backfill 鍘嗗彶鏁版嵁
   - 鏌ヨ鏃舵棤闇€鍔ㄦ€佽绠楋紝鎬ц兘鏇村ソ

### 8.4 鏉ユ簮涓嬮捇璺緞

```
Sources 鍒楄〃
    鈫?    鈹溾攢 鐐瑰嚮 "Google"
    鈹?   鈹斺攢 Search Console API 鈫?鍏抽敭璇嶅垪琛?    鈹?    鈹斺攢 鐐瑰嚮鍏朵粬鏉ユ簮
         鈹斺攢 /referrers/:referrer 鈫?鍏蜂綋 referrer 鍩熷悕鍒楄〃
              鈫?              鈹斺攢 搴旂敤杩囨护鍣?"referrer is X" 鍒颁华琛ㄧ洏
```

### 8.5 鍏抽敭鎶€鏈€夋嫨

- **鍦扮悊浣嶇疆**锛歁axMind/DB-IP MMDB + locus 搴?- **璁惧璇嗗埆**锛歎AInspector 搴?+ 200ms 瓒呮椂淇濇姢 + 缂撳瓨
- **鏉ユ簮璇嗗埆**锛歊efInspector 搴?+ 鑷畾涔夎鍒?+ UTM 鍙傛暟
- **娓犻亾鎺ㄦ柇**锛欳lickHouse MATERIALIZED 鍒?+ SQL 鍑芥暟 + Dictionary

