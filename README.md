# Working script for shorter message to slack


var SEVERITY_COLORS = [
    '#97AAB3', '#7499FF', '#FFC859',
    '#FFA059', '#E97659', '#E45959'
];

var RESOLVE_COLOR = '#009900';

var SLACK_MODE_HANDLERS = {
    alarm: handlerAlarm,
    event: handlerEvent
};


if (!String.prototype.format) {
    String.prototype.format = function() {
        var args = arguments;

        return this.replace(/{(\d+)}/g, function(match, number) {
            return number in args
                ? args[number]
                : match
            ;
        });
    };
}

function isEventProblem(params) {
    return params.event_value == 1
        && params.event_update_status == 0
    ;
}

function isEventUpdate(params) {
    return params.event_value == 1
        && params.event_update_status == 1
    ;
}

function isEventResolve(params) {
    return params.event_value == 0;
}

function getPermalink(channelId, messageTimestamp) {
    var req = new HttpRequest();

    if (typeof params.HTTPProxy === 'string' && params.HTTPProxy.trim() !== '') {
        req.setProxy(params.HTTPProxy);
    }

    req.addHeader('Content-Type: application/x-www-form-urlencoded; charset=utf-8');
    req.addHeader('Authorization: Bearer ' + params.bot_token);

    var query = '{0}?channel={1}&message_ts={2}'.format(
            Slack.getPermalink,
            encodeURIComponent(channelId),
            encodeURIComponent(messageTimestamp)),
        resp = JSON.parse(req.get(query));

    if (req.getStatus() != 200 || !resp.ok || resp.ok === 'false') {
        throw 'message was created, but getting message link was failed with reason "' + resp.error + '"';
    }

    return resp.permalink;
}

function createProblemURL(zabbix_url, triggerid, eventid, event_source) {
    var problem_url = '';
    if (event_source === '0') {
        problem_url = '{0}/tr_events.php?triggerid={1}&eventid={2}'
            .format(
                zabbix_url,
                triggerid,
                eventid
            );
    }
    else {
        problem_url = zabbix_url;
    }

    return problem_url;
}

function handlerAlarm(params) {
    var fields = {
        channel: params.channel,
        as_user: params.slack_as_user,
    };

    if (isEventProblem(params)) {
        fields.attachments = [
            createMessage(
                SEVERITY_COLORS[params.event_nseverity] || 0,
                params.event_date,
                params.event_time,
                createProblemURL(params.zabbix_url, params.trigger_id, params.event_id, params.event_source)
            )
        ];

        var resp = JSON.parse(req.post(Slack.postMessage, JSON.stringify(fields)));

        if (req.getStatus() != 200 || !resp.ok || resp.ok === 'false') {
            throw resp.error;
        }

        result.tags = {
            ['__message_ts_' + params.channel]: resp.ts,
            ['__channel_id_' + params.channel]: resp.channel,
            ['__message_link_' + params.channel]: getPermalink(resp.channel, resp.ts),
        };

    }
    else if (isEventUpdate(params)) {
        try {
            var channel_event_tags = JSON.parse(params.event_tags);
        } catch (error) {
            throw 'Cannot process event tags: ' + error;
        }

        if (Array.isArray(channel_event_tags)) {
            for (i in channel_event_tags) {
                if (channel_event_tags[i].tag.includes('__message_ts_' + params.channel)) {
                    fields.thread_ts = channel_event_tags[i].value;
                    break;
                }
            }
        }

        fields.attachments = [
            createMessage(
                SEVERITY_COLORS[params.event_nseverity] || 0,
                params.event_update_date,
                params.event_update_time,
                createProblemURL(params.zabbix_url, params.trigger_id, params.event_id, params.event_source),
                true
            )
        ];

        resp = JSON.parse(req.post(Slack.postMessage, JSON.stringify(fields)));

        if (req.getStatus() != 200 || !resp.ok || resp.ok === 'false') {
            throw resp.error;
        }

    }
    else if (isEventResolve(params)) {

        fields.text = '';

        try {
            var channel_event_tags = JSON.parse(params.event_tags);
        } catch (error) {
            throw 'Cannot process event tags: ' + error;
        }

        if (Array.isArray(channel_event_tags)) {
            for (i in channel_event_tags) {
                if (channel_event_tags[i].tag.includes('__channel_id_' + params.channel)) {
                    fields.channel = channel_event_tags[i].value;
                    continue;
                }
                if (channel_event_tags[i].tag.includes('__message_ts_' + params.channel)) {
                    fields.ts = channel_event_tags[i].value;
                }
            }
        }

        fields.attachments = [
            createMessage(
                RESOLVE_COLOR,
                params.event_date,
                params.event_time,
                createProblemURL(params.zabbix_url, params.trigger_id, params.event_id, params.event_source)
            )
        ];

        resp = JSON.parse(req.post(Slack.chatUpdate, JSON.stringify(fields)));
        if (req.getStatus() != 200 || !resp.ok || resp.ok === 'false') {
            throw resp.error;
        }
    }
}

function handlerEvent(params) {
    var fields = {
        channel: params.channel,
        as_user: params.slack_as_user
    };

    if (isEventProblem(params)) {
        fields.attachments = [
            createMessage(
                SEVERITY_COLORS[params.event_nseverity] || 0,
                params.event_date,
                params.event_time,
                createProblemURL(params.zabbix_url, params.trigger_id, params.event_id, params.event_source)
            )
        ];

        var resp = JSON.parse(req.post(Slack.postMessage, JSON.stringify(fields)));

        if (req.getStatus() != 200 || !resp.ok || resp.ok === 'false') {
            throw resp.error;
        }

        result.tags = {
            ['__message_link_' + params.channel]: getPermalink(resp.channel, resp.ts)
        }

    }
    else if (isEventUpdate(params)) {
        fields.attachments = [
            createMessage(
                SEVERITY_COLORS[params.event_nseverity] || 0,
                params.event_update_date,
                params.event_update_time,
                createProblemURL(params.zabbix_url, params.trigger_id, params.event_id, params.event_source),
                false
            )
        ];

        resp = JSON.parse(req.post(Slack.postMessage, JSON.stringify(fields)));

        if (req.getStatus() != 200 || !resp.ok || resp.ok === 'false') {
            throw resp.error;
        }

    }
    else if (isEventResolve(params)) {
        fields.attachments = [
            createMessage(
                RESOLVE_COLOR,
                params.event_recovery_date,
                params.event_recovery_time,
                createProblemURL(params.zabbix_url, params.trigger_id, params.event_id, params.event_source)
            )
        ];

        resp = JSON.parse(req.post(Slack.postMessage, JSON.stringify(fields)));

        if (req.getStatus() != 200 || !resp.ok || resp.ok === 'false') {
            throw resp.error;
        }
    }
}

function createMessage(
    event_severity_color,
    event_date,
    event_time,
    problem_url,
    isShort,
    messageText
) {
    var message = {
        fallback: params.alert_subject,
        title: params.alert_subject,
        color: event_severity_color,
        title_link: problem_url,
        pretext: messageText || '',

        fields: [
            {
                title: 'Host',
                value: '{0} [{1}]'.format(params.host_name, params.host_conn),
                short: true
            },
            {
                title: 'Event time',
                value: '{0} {1}'.format(event_date, event_time),
                short: true
            }
        ],
    };

    if (params.event_source === '0') {
        message.fields.push(
            {
                title: 'Opdata',
                value: params.event_opdata,
                short: true
            }
        );
    }

    if (!isShort  && params.event_source === '0') {
        message['actions'] = [
            {
                type: 'button',
                text: 'Open in Zabbix',
                url: problem_url
            }
        ];

        message.fields.push(
            {

            }
        );
    }

    return message;
}

function validateParams(params) {
    if (typeof params.bot_token !== 'string' || params.bot_token.trim() === '') {
        throw 'Field "bot_token" cannot be empty';
    }

    if (typeof params.channel !== 'string' || params.channel.trim() === '') {
        throw 'Field "channel" cannot be empty';
    }

    if (isNaN(params.event_id)) {
        throw 'Field "event_id" is not a number';
    }

    if ([0, 1, 2, 3].indexOf(parseInt(params.event_source)) === -1) {
        throw 'Incorrect "event_source" parameter given: "' + params.event_source + '".\nMust be 0-3.';
    }

    if (params.event_source !== '0') {
        params.event_nseverity = '0';
        params.event_severity = 'Not classified';
        params.event_update_status = '0';
        params.slack_mode = 'event';
    }

    if (params.event_source === '1' || params.event_source === '2') {
        params.event_value = '1';
    }

    if (params.event_source === '1') {
        params.host_name = params.discovery_host_dns;
        params.host_ip = params.discovery_host_ip;
    }

    if (!~[0, 1, 2, 3, 4, 5].indexOf(parseInt(params.event_nseverity))) {
        throw 'Incorrect "event_nseverity" parameter given: ' + params.event_nseverity + '\nMust be 0-5.';
    }

    if (typeof params.event_severity !== 'string' || params.event_severity.trim() === '') {
        throw 'Field "event_severity" cannot be empty';
    }

    if (params.event_update_status !== '0' && params.event_update_status !== '1') {
        throw 'Incorrect "event_update_status" parameter given: ' + params.event_update_status + '\nMust be 0 or 1.';
    }

    if (params.event_value !== '0' && params.event_value !== '1') {
        throw 'Incorrect "event_value" parameter given: ' + params.event_value + '\nMust be 0 or 1.';
    }

    if (typeof params.host_conn !== 'string' || params.host_conn.trim() === '') {
        throw 'Field "host_conn" cannot be empty';
    }

    if (typeof params.host_name !== 'string' || params.host_name.trim() === '') {
        throw 'Field "host_name" cannot be empty';
    }

    if (!~['true', 'false'].indexOf(params.slack_as_user.toLowerCase())) {
        throw 'Incorrect "slack_as_user" parameter given: ' + params.slack_as_user + '\nMust be "true" or "false".';
    }

    if (!~['alarm', 'event'].indexOf(params.slack_mode)) {
        throw 'Incorrect "slack_mode" parameter given: ' + params.slack_mode + '\nMust be "alarm" or "event".';
    }

    if (isNaN(params.trigger_id) && params.event_source === '0') {
        throw 'field "trigger_id" is not a number';
    }

    if (typeof params.zabbix_url !== 'string' || params.zabbix_url.trim() === '') {
        throw 'Field "zabbix_url" cannot be empty';
    }

    if (!/^(http|https):\/\/.+/.test(params.zabbix_url)) {
        throw 'Field "zabbix_url" must contain a schema';
    }
}

try {
    var params = JSON.parse(value);

    validateParams(params);

    var req = new HttpRequest(),
        result = {tags: {}};

    if (typeof params.HTTPProxy === 'string' && params.HTTPProxy.trim() !== '') {
        req.setProxy(params.HTTPProxy);
    }

    req.addHeader('Content-Type: application/json; charset=utf-8');
    req.addHeader('Authorization: Bearer ' + params.bot_token);

    var slack_endpoint = 'https://slack.com/api/';

    var Slack = {
        postMessage: slack_endpoint + 'chat.postMessage',
        getPermalink: slack_endpoint + 'chat.getPermalink',
        chatUpdate: slack_endpoint + 'chat.update'
    };

    params.slack_mode = params.slack_mode.toLowerCase();
    params.slack_mode = params.slack_mode in SLACK_MODE_HANDLERS
        ? params.slack_mode
        : 'alarm';

    SLACK_MODE_HANDLERS[params.slack_mode](params);

    if (params.event_source === '0') {
        return JSON.stringify(result);
    }
    else {
        return 'OK';
    }
}
catch (error) {
    Zabbix.log(4, '[ Slack Webhook ] Slack notification failed : ' + error);
    throw 'Slack notification failed : ' + error;
}
