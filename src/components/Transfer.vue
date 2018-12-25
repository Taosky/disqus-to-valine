<template>
    <div>
        <el-row :gutter="20">
            <el-col :span="12" :offset="6" v-loading="loading"
                    :element-loading-text="finishedNumber + '/' + total"
                    element-loading-spinner="el-icon-loading">
                <div style="text-align: left">
                    <p style="font-size: 28px;">说明
                    <p>
                    <p>1. 用于现有Disqus评论转移至Valine(LeanCloud)。</p>
                    <p>2. 回复楼层、时间显示正常；由于Disqus导出内容限制，邮箱、头像、网站链接丢失。</p>
                    <p>3. 需要临时将 <span style="font-weight: bold;color: #e11212cc;">https://valine.mou.science</span> 添加到<span>LeanCloud安全域名</span>。</p>
                    <p>4. 数据会多出一个DisqusToValineTest，是检查ID和KEY用的，可以删除。</p>
                    <p>5. 纯前端操作，无风险，项目代码：https://github.com/taosky/disqus-to-valine</p>

                </div>
                <div style="margin-bottom: 20px;">
                    <el-input v-model="leanId" placeholder="LeanCloud APP ID"></el-input>
                    <el-input v-model="leanKey" placeholder="LeanCloud APP KEY"></el-input>
                    <el-input
                            type="textarea"
                            :rows="20"
                            placeholder="Disqus导出的XML内容"
                            v-model="disqusXml">
                    </el-input>
                </div>
                <div>
                    <el-button type="warning" @click="startTrans">开始转换</el-button>
                </div>
            </el-col>
        </el-row>


    </div>
</template>

<script>
    window.convert = require('xml-js');
    import axios from 'axios';

    export default {
        name: "transfer",
        data: function () {
            return {
                leanId: '',
                leanKey: '',
                disqusXml: '',
                threads: {},
                comments: {},
                sortedComments: [],
                finishedNumber: 0,
                total: 0,
                loading: false
            }
        },
        methods: {
            sendNotify: function (msg) {
                this.$notify({
                    title: '提示',
                    message: msg,
                    type: 'success',
                    duration: 0,
                });

            },
            sendError: function (msg) {
                this.$notify.error({
                    title: '错误',
                    message: msg,
                    duration: 5000,
                });

            },
            xml2js: function () {
                let disqusJs = {};
                try {
                    disqusJs = window.convert.xml2js(this.disqusXml, {compact: true});
                }
                catch (err) {
                    this.sendError('XML格式错误');
                    return null
                }
                if (disqusJs.hasOwnProperty('disqus') && disqusJs['disqus']['_attributes']['xmlns'] === 'http://disqus.com') {
                    return disqusJs
                }
                this.sendError('XML格式错误');
                return null
            },
            reUrl: function (url) {
                if (!url) {
                    return url;
                }
                let format_url = url;
                if (format_url.indexOf('index.html')) {
                    format_url = format_url.replace('index.html', '').replace('//', '/');
                }
                if (!format_url.startsWith('/')) {
                    format_url = '/' + format_url;
                }
                if (!format_url.endsWith('/')) {
                    format_url = format_url + '/';
                }
                return format_url;
            },
            sendLean: function (post, comment, that) {
                let api = `https://${that.leanId.slice(0, 8)}.api.lncld.net/1.1/classes/Comment`;
                let headers = {
                    'X-LC-Id': that.leanId, 'X-LC-Key': that.leanKey,
                    'Content-Type': 'application/json'
                };
                if (post.hasOwnProperty('parent')) {
                    let parentId = post['parent']['_attributes']['dsq:id'];
                    let parentObjectId = that.comments[parentId]['objectId'];
                    comment['pid'] = parentObjectId;
                    comment['rid'] = parentObjectId;
                }
                return new Promise((resolve, reject) => {
                    axios.post(api, comment, {headers: headers}).then(response => {
                        if (response.data) {
                            resolve(response.data)
                        } else {
                            reject(response)
                        }
                    }).catch(error => {
                        reject(error);
                    });
                })
            },
            startTrans: function () {
                let that = this;
                let disqusJs = this.xml2js();
                // check xml
                if (disqusJs === null) {
                    return;
                }
                that.total = disqusJs.disqus['post'].length;
                that.loading = true;
                // read threads
                disqusJs.disqus['thread'].forEach(function (thread) {
                    if (thread.id.hasOwnProperty('_text')) {
                        that.threads[thread._attributes['dsq:id']] = that.reUrl(thread.id._text);
                    }
                });
                // read comments
                for (let post of disqusJs.disqus['post']) {
                    let status = 1;
                    if (post['isDeleted']['_text'] !== 'false' || post['isSpam']['_text'] !== 'false') {
                        status = 0;
                    }
                    that.comments[post._attributes['dsq:id']] = {
                        'status': status,
                        'nick': post['author']['name']['_text'],
                        'url': that.threads[post['thread']['_attributes']['dsq:id']],
                        'comment': post['message']['_cdata'],
                        'insertedAt': {"__type": "Date", "iso": post['createdAt']['_text'].replace('Z', '.249Z')},
                    };
                }

                let requestList = [];
                let sortedComments = disqusJs.disqus['post'].sort((a, b) => a._attributes['dsq:id'] - b._attributes['dsq:id']);
                sortedComments.forEach(function (post) {
                    let comment = that.comments[post._attributes['dsq:id']];
                    requestList.push({'post': post, 'comment': comment});

                });
                let api = `https://${that.leanId.slice(0, 8)}.api.lncld.net/1.1/classes/DisqusToValineTest`;
                let headers = {
                    'X-LC-Id': that.leanId, 'X-LC-Key': that.leanKey,
                    'Content-Type': 'application/json'
                };
                axios.post(api, {'title': 'disqus-to-valine'}, {headers: headers}).then(function () {
                    // send requests
                    requestList.reduce((previousValue, currentValue) => {
                        return previousValue.then(() => that.sendLean(currentValue['post'], currentValue['comment'], that).then(function (data) {
                            that.comments[currentValue['post']._attributes['dsq:id']]['objectId'] = data.objectId;
                            that.finishedNumber = that.finishedNumber + 1;
                            if (that.total === that.finishedNumber) {
                                that.loading = false;
                                that.sendNotify(`转移完毕，共${that.total}条。`)
                            }
                        }));
                    }, Promise.resolve());
                }).catch(function () {
                    that.loading = false;
                    that.sendError('请检查LeanCloud ID和KEY')
                });

            }
        }
    }
</script>
<style scoped>

</style>